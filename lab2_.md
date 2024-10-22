# Лабораторная работа №2
  
Кузьмин Василий пм-42
  
## Задание 2. Реализация задачи численного интегрирования по гибридной технологии
  
### На своем компьютере
  
```bash
admin1@Ubuntu:~/parallel$ ./ex04c.px 1 1
  
mt=0 a1=0.000000e+00 b1=1.000000e+00 n1=1000000000 s1=3.141593e+00
  
time=10.390402 sum= 3.141592651589658e+00
  
admin1@Ubuntu:~/parallel$ ./ex04c.px 2 1
  
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00
  
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00
  
time=5.360441 sum= 3.141592648389766e+00
  
admin1@Ubuntu:~/parallel$ ./ex04c.px 1 2
  
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00
  
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00
  
time=5.218593 sum= 3.141592648389766e+00
  
admin1@Ubuntu:~/parallel$ ./ex04c.px 2 2
  
mt=2 a1=5.000000e-01 b1=7.500000e-01 n1=250000000 s1=7.194140e-01
  
mt=0 a1=0.000000e+00 b1=2.500000e-01 n1=250000000 s1=9.799146e-01
  
mt=3 a1=7.500000e-01 b1=1.000000e+00 n1=250000000 s1=5.675882e-01
  
mt=1 a1=2.500000e-01 b1=5.000000e-01 n1=250000000 s1=8.746758e-01
  
time=2.630080 sum= 3.141592642065225e+00
```
### На сервере
```bash
miet029@sr210:~/work$ ./ex04c.px 1 1
  
mt=0 a1=0.000000e+00 b1=1.000000e+00 n1=1000000000 s1=3.141593e+00
  
time=2.789779 sum= 3.141592651589658e+00
  
miet029@sr210:~/work$ ./ex04c.px 2 1
  
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00
  
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00
  
time=1.447228 sum= 3.141592648389766e+00
  
miet029@sr210:~/work$ ./ex04c.px 1 2
  
mt=1 a1=5.000000e-01 b1=1.000000e+00 n1=500000000 s1=1.287002e+00
  
mt=0 a1=0.000000e+00 b1=5.000000e-01 n1=500000000 s1=1.854590e+00
  
time=1.442571 sum= 3.141592648389766e+00
  
miet029@sr210:~/work$ ./ex04c.px 2 2
  
mt=3 a1=7.500000e-01 b1=1.000000e+00 n1=250000000 s1=5.675882e-01
  
mt=2 a1=5.000000e-01 b1=7.500000e-01 n1=250000000 s1=7.194140e-01
  
mt=1 a1=2.500000e-01 b1=5.000000e-01 n1=250000000 s1=8.746758e-01
  
mt=0 a1=0.000000e+00 b1=2.500000e-01 n1=250000000 s1=9.799146e+00
  
time=0.795642 sum= 3.141592642065225e+00
  
```
### Код программы
```C
#include <math.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <sys/types.h>
#include <time.h>
#include <unistd.h>
  
#include "mycom.h"
#define MSG_ID 7777
#define MSG_PERM 00600
#define LBUF 20
  
typedef struct tag_msg_t {
  int n;
  double buf[LBUF];
} msg_t;
  
static pthread_t* threads;
static double sums[LBUF];
static int msgid;
static msg_t msg;
static int np, mp, tp;
static double a = 0;
static double b = 1;
static int ni = 1000000000;
  
pid_t NetInit(int np, int* mp);
double f1(double x);
double f1(double x) { return 4.0 / (1.0 + x * x); }
void* myjobt(void* m);
  
void* myjobt(void* m) {
  int n1, mt = (int)(((long long int)m) % 1024);
  double a1, b1, h1;
  
  n1 = ni / (np * tp);
  h1 = (b - a) / (np * tp);
  a1 = a + h1 * mt;
  
  if (mt < np * tp - 1)
    b1 = a1 + h1;
  else
    b1 = b;
  
  sums[mt] = integrate(f1, a1, b1, n1);
  printf("mt=%d a1=%le b1=%le n1=%d s1=%le\n", mt, a1, b1, n1, sums[mt]);
  
  return 0;
}
  
int main(int argc, char* argv[]) {
  double t;
  double sum = 0;
  pid_t spid;
  
  if (argc < 3) {
    printf("Usage: %s <process number>\n", argv[0]);
    return 1;
  }
  
  sscanf(argv[1], "%d", &np);
  sscanf(argv[2], "%d", &tp);
  
  mp = 0;
  t = mytime(0);
  
  if (!(threads = (pthread_t*)malloc(np * tp * sizeof(pthread_t)))) {
    perror("Error allocating memory for threads");
  }
  
  spid = NetInit(np, &mp);
  
  if (spid > 0) {
    msgid = msgget(MSG_ID, MSG_PERM | IPC_CREAT);
    for (int i = 1; i < np; i++) {
      msg.n = i;
      msgsnd(msgid, &msg, sizeof(msg_t), 0);
    }
    msg.n = 0;
  } else if (np > 1) {
    while ((msgid = msgget(MSG_ID, MSG_PERM)) < 0);
    msgrcv(msgid, &msg, sizeof(msg_t), 0, 0);
  }
  
  for (int i = 0; i < tp; i++) {
    if (pthread_create(threads + tp * msg.n + i, 0, myjobt, (void*)(tp * msg.n + i))) {
      printf("nothing\n");
    }
  }
  
  for (int i = 0; i < tp; i++) {
    if (pthread_join(threads[tp * msg.n + i], 0)) {
    }
  }
  
  if (spid == 0) {
    for (int i = 0; i < tp; i++) {
      sum = sum + sums[tp * msg.n + i];
    }
  
    if (np > 1) {
      msg.buf[0] = sum;
      msgsnd(msgid, &msg, sizeof(msg_t), 0);
    }
  }
  
  if (spid > 0) {
    while (wait(NULL) > 0);
    if ((spid > 0)) {
      for (int i = 1; i < np; i++) {
        msgrcv(msgid, &msg, sizeof(msg_t), 0, 0);
        sum += msg.buf[0];
      }
      msgctl(msgid, IPC_RMID, (struct msqid_ds*)0);
    }
  
    for (int i = 0; i < tp; i++) {
      sum = sum + sums[i];
    }
  
    t = mytime(1);
    printf("time=%lf sum=%22.15le\n", t, sum);
  } else if (spid == 0 && np < 2) {
    t = mytime(1);
    printf("time=%lf sum=%22.15le\n", t, sum);
  }
  
  return 0;
}
  
pid_t NetInit(int np, int* mp) {
  int i;
  pid_t spid = 0;
  
  if (np > 1) {
    i = 1;
    while (i < np) {
      if (spid > 0 || i == 1) {
        *mp = i;
        spid = fork();
      }
      if (spid == 0) return 0;
      i++;
    }
  }
  
  *mp = 0;
  return spid;
}
```
  