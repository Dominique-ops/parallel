# Лабораторная работа №3

Кузьмин Василий пм-42

## Задание 3. Использование семафоров и общих сегментов памяти

### Компиляция 
```bash
mpicc -o ex05d.px -O2 -fopenmp ex05d.c mycom.c -lm
```

### На своем компьютере

```bash
admin1@Ubuntu:~/parallel$ ./ex05d.px 1 1                                                                                                                                 
time=10.914408 sum= 3.141592651589658e+00                                                                                                                                
admin1@Ubuntu:~/parallel$ ./ex05d.px 2 1                                                                                                                                 
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00                                                                                                        
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00                                                                                                        
time=5.632504 sum= 3.141592648389766e+00                                                                                                                                 
admin1@Ubuntu:~/parallel$ ./ex05d.px 1 2                                                                                                                                 
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00                                                                                                        
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00                                                                                                        
time=5.595312 sum= 3.141592648389766e+00                                                                                                                                 
admin1@Ubuntu:~/parallel$ ./ex05d.px 2 2                                                                                                                                 
mt=2 a1=5.000000e-01 b1=7.500000e-01 n1=250000000 s1=7.194140e-01                                                                                                        
mt=3 a1=7.500000e-01 b1=1.000000e+00 n1=250000000 s1=5.675882e-01                                                                                                        
mt=1 a1=2.500000e-01 b1=5.000000e-01 n1=250000000 s1=8.746758e-01                                                                                                        
mt=0 a1=0.000000e+00 b1=2.500000e-01 n1=250000000 s1=9.799146e-01                                                                                                        
time=2.767734 sum= 3.141592642065224e+00
```
### На сервере
```bash
miet029@sr210:~/work$ ./ex05d.px 1 1                                                                                                                                     
time=2.792958 sum= 3.141592651589658e+00                                                                                                                                 
miet029@sr210:~/work$ ./ex05d.px 2 1                                                                                                                                     
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00                                                                                                        
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00                                                                                                        
time=1.444912 sum= 3.141592648389766e+00                                                                                                                                 
miet029@sr210:~/work$ ./ex05d.px 1 2                                                                                                                                     
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00                                                                                                        
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00                                                                                                        
time=1.429783 sum= 3.141592648389766e+00                                                                                                                                 
miet029@sr210:~/work$ ./ex05d.px 2 2                                                                                                                                     
mt=0 a1=0.000000e+00 b1=2.500000e-01 n1=250000000 s1=9.799146e-01                                                                                                        
mt=2 a1=5.000000e-01 b1=7.500000e-01 n1=250000000 s1=7.194140e-01                                                                                                        
mt=1 a1=2.500000e-01 b1=5.000000e-01 n1=250000000 s1=8.746758e-01                                                                                                        
mt=3 a1=7.500000e-01 b1=1.000000e+00 n1=250000000 s1=5.675882e-01                                                                                                        
time=0.792633 sum= 3.141592642065224e+00 
```
### Код программы
```C
#include <math.h>
#include <omp.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/types.h>
#include <unistd.h>

#include "mycom.h"

#define SEM_ID 2007
#define SHM_ID 2008
#define PERMS 0600

typedef struct tag_msg_t {
  int n;
  double s;
} msg_t;

#ifdef _SEM_SEMUN_UNDEFINED
union semun {
  int val;
  struct semid_ds *buf;
  unsigned short int *array;
  struct seminfo *__buf;
};
#endif

int np, mp, tp;
int semid;
int shmid;
msg_t *msg;
struct sembuf sem_loc = {0, -1, 0};
struct sembuf sem_unloc = {0, 1, 0};
union semun s_un;

double a = 0;
double b = 1;
int ni = 1000000000;
int num = 0;
double sum = 0;

void myjobt(int mt);

double f1(double x);
double f1(double x) { return 4.0 / (1.0 + x * x); }

void parallel();

void parallel() {
#pragma omp parallel
  {
    int mt = omp_get_thread_num();

    myjobt(tp * mp + mt);

  }  // end parallel

  while (num < tp);
}

void myjobt(int mt) {
  int n1;
  double a1, b1, h1, s, n = np * tp;

  n1 = ni / n;

  h1 = (b - a) / n;

  a1 = a + h1 * mt;

  if (mt < n - 1)
    b1 = a1 + h1;
  else
    b1 = b;

  s = integrate(f1, a1, b1, n1);

  printf("mt=%d a1=%le b1=%le n1=%d s1=%le\n", mt, a1, b1, n1, s);

#pragma omp critical
  {
    sum += s;
    num++;

  }  // end critical

  return;
}

void NetInit(int np, int *mp);
void NetInit(int np, int *mp) {
  int i;
  pid_t spid = 0;

  if (np > 1) {
    *mp = 1;
    spid = fork();
    if (spid > 0 && np > 2)
      for (i = 2; i < np; i++)
        if (spid > 0) {
          *mp = i;
          spid = fork();
        }
    if (spid > 0) *mp = 0;
  } else
    *mp = 0;

  return;
}

void ShrMemInit();
void ShrMemInit() {
  if (mp == 0) {
    if ((shmid = shmget(SHM_ID, sizeof(msg_t), PERMS | IPC_CREAT)) < 0)
      if ((shmid = shmget(SHM_ID, sizeof(msg_t), PERMS)) < 0)
        myerr("Can not find shared memory segment", 1);

    if ((msg = (msg_t *)shmat(shmid, 0, 0)) == NULL)
      myerr("Can not attach shared memory segment", 2);

    msg->n = 0;
    msg->s = 0;  // initialization of shared memory

    if ((semid = semget(SEM_ID, 1, PERMS | IPC_CREAT)) < 0)
      if ((semid = semget(SEM_ID, 1, PERMS)) < 0)
        myerr("Can not find semaphore", 3);

    s_un.val = 1;
    semctl(semid, 0, SETVAL, s_un);  // unlock
  } else {
    while ((shmid = shmget(SHM_ID, sizeof(msg_t), PERMS)) < 0);

    if ((msg = (msg_t *)shmat(shmid, 0, 0)) == NULL)
      myerr("Can not attach shared memory segment", 4);

    while ((semid = semget(SEM_ID, 1, PERMS)) < 0);
  }

  return;
}

void ShrMemDone();
void ShrMemDone() {
  if (mp == 0) {
    if (semctl(semid, 0, IPC_RMID, (struct semid_ds *)0) < 0)
      myerr("Can not remove semaphore", 5);

    if (shmdt(msg) < 0) myerr("Can not dettach shared memory segment", 6);

    if (shmctl(shmid, IPC_RMID, (struct shmid_ds *)0) < 0)
      myerr("Can not remove shared memory segment", 7);
  } else {
    if (shmdt(msg) < 0) myerr("Can not dettach shared memory segment", 8);
    exit(0);
  }

  return;
}

int main(int argc, char *argv[]) {
  double t;

  if (argc < 3) {
    printf("Usage: %s <process number>\n", argv[0]);
    return 1;
  }

  sscanf(argv[1], "%d", &np);
  sscanf(argv[2], "%d", &tp);

  if (np < 1) np = 1;
  mp = 0;

  t = mytime(0);

  if (np < 2) {
    if (tp < 2)
      sum = integrate(f1, a, b, ni);
    else {
      omp_set_num_threads(tp);

      parallel();
    }
  } else {
    int n1;
    NetInit(np, &mp);

    if (np > 1) ShrMemInit();

    omp_set_num_threads(tp);

    parallel();

    while (semop(semid, &sem_loc, 1) < 0);  // wait + lock

    msg->n++;
    msg->s += sum;

    while (semop(semid, &sem_unloc, 1) < 0);  // wait + unlock

    if (mp == 0) {
      n1 = 0;
      while (n1 < np) {
        while (semop(semid, &sem_loc, 1) < 0);  // wait + lock
        n1 = msg->n;
        sum = msg->s;
        while (semop(semid, &sem_unloc, 1) < 0);  // wait + unlock
      }
    }

    if (np > 1) ShrMemDone();
  }

  t = mytime(1);

  printf("time=%lf sum=%22.15le\n", t, sum);

  return 0;
}
```