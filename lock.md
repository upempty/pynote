- basic idea for locking
```
typedef struct {
  int lock; //1 means locked by.
  // other fields.
} p_lock_t;

int p_to_lock(p_lock_t *p) {
  while （true) {
  if (compare_and_swap(&p->lock, 0, 1) == 0) {
  return 0;
  }
  else {
  block_current_thread();// when lock is release and this thread/process be wokenup, continue to loop try to accquire the lock.
  }
  }
}
```
