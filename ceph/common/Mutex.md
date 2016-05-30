# Ceph中的Mutex操作

    class Mutex {
    private:
      std::string name;
      int id;
      bool recursive;//递归锁，
      bool lockdep;
      bool backtrace;  // gather backtrace on lock acquisition

      pthread_mutex_t _m; //POSIX互斥量
      int nlock; //锁被持有的数目
      pthread_t locked_by; //加锁的进程
      CephContext *cct;
      PerfCounters *logger;

      // don't allow copying.
      void operator=(const Mutex &M);
      Mutex(const Mutex &M);

      void _register() {
        id = lockdep_register(name.c_str());
      }
      void _will_lock() { // about to lock
        id = lockdep_will_lock(name.c_str(), id, backtrace);
      }
      void _locked() {    // just locked
        id = lockdep_locked(name.c_str(), id, backtrace);
      }
      void _will_unlock() {  // about to unlock
        id = lockdep_will_unlock(name.c_str(), id);
      }

    public:
      Mutex(const std::string &n, bool r = false, bool ld=true, bool bt=false,
    	CephContext *cct = 0);
      ~Mutex();
      bool is_locked() const {
        return (nlock > 0);
      }
      bool is_locked_by_me() const {
        return nlock > 0 && locked_by == pthread_self();
      }

      bool TryLock() {
        int r = pthread_mutex_trylock(&_m);
        if (r == 0) {
          if (lockdep && g_lockdep) _locked();
          _post_lock();
        }
        return r == 0;
      }

      void Lock(bool no_lockdep=false);

      void _post_lock() {
        if (!recursive) {
          assert(nlock == 0);
          locked_by = pthread_self();
        };
        nlock++;
      }

      void _pre_unlock() {
        assert(nlock > 0);
        --nlock;
        if (!recursive) {
          assert(locked_by == pthread_self());
          locked_by = 0;
          assert(nlock == 0);
        }
      }
      void Unlock();

      friend class Cond;


    public:
      class Locker {
        Mutex &mutex;

      public:
        explicit Locker(Mutex& m) : mutex(m) {
          mutex.Lock();
        }
        ~Locker() {
          mutex.Unlock();
        }
      };
    };
