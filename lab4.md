# Лабораторная работа №4

Кузьмин Василий пм-42

## Задание 3. Решение задачи численного интегрирования в сетевой модели

### Компиляция 
```bash
mpicc -o ex07b.px -O2 ex07b.c mycom.c mynet.c -lm -fopenmp
```
### Запуск
```bash
mpirun -np 2 ex07b.px 2
```

### На своем компьютере

```bash
admin1@Ubuntu:~/parallel$ mpirun -np 1 ex07b.px 1                                     
Netsize: 1, process: 0, system: Ubuntu, tick=1.000000e-09                                                                                                                   
mp=0 t1=10.620938 t2=0.000000 t3=10.620938 int= 3.141592651589658e+00 
admin1@Ubuntu:~/parallel$ mpirun  -np 4 ex07b.px 4                                  
Netsize: 4, process: 2, system: Ubuntu, tick=1.000000e-09                             
Netsize: 4, process: 1, system: Ubuntu, tick=1.000000e-09                             
Netsize: 4, process: 0, system: Ubuntu, tick=1.000000e-09                             
Netsize: 4, process: 3, system: Ubuntu, tick=1.000000e-09                             
mp=1 t1=1.823046 t2=0.028315 t3=1.851361 int= 8.746757697874445e-01                   
mp=2 t1=1.789185 t2=0.062174 t3=1.851358 int= 1.287002197591606e+00                   
mp=0 t1=1.815403 t2=0.035961 t3=1.851364 int= 3.141592604334757e+00                   
mp=3 t1=1.845156 t2=0.006203 t3=1.851360 int= 5.675882096128860e-01  
```
### На сервере
```bash
miet029@sr210:~/work$ mpirun -np 1 ex07b.px 1                                         
Authorization required, but no authorization protocol specified                       
Authorization required, but no authorization protocol specified                       
Netsize: 1, process: 0, system: sr210, tick=1.000000e-09                              
mp=0 t1=2.765069 t2=0.000000 t3=2.765069 int= 3.141592651589658e+00
miet029@sr210:~/work$ mpirun -np 4 ex07b.px 4                                       
Authorization required, but no authorization protocol specified                       
Authorization required, but no authorization protocol specified                       
Netsize: 4, process: 0, system: sr210, tick=1.000000e-09                              
Netsize: 4, process: 1, system: sr210, tick=1.000000e-09                              
Netsize: 4, process: 2, system: sr210, tick=1.000000e-09                              
Netsize: 4, process: 3, system: sr210, tick=1.000000e-09                              
mp=2 t1=0.264159 t2=0.000020 t3=0.264178 int= 1.287002197591606e+00                   
mp=3 t1=0.263723 t2=0.000457 t3=0.264180 int= 5.675882096128860e-01                   
mp=0 t1=0.263690 t2=0.000499 t3=0.264189 int= 3.141592604334757e+00                   
mp=1 t1=0.263607 t2=0.000573 t3=0.264179 int= 8.746757697874445e-01
```
### Код программы
```C
#include <math.h>
#include <mpi.h>
#include <omp.h>
#include <stdio.h>

#include "mycom.h"
#include "mynet.h"
int np, mp, nl, tp;
char pname[MPI_MAX_PROCESSOR_NAME];
MPI_Status status;
static double tick, t1, t2, t3;
double a = 0;
static double b = 1;
int ni = 1000000000;
double sum = 0;
int num = 0;
double f1(double x);
double f1(double x) { return 4.0 / (1.0 + x * x); }

void myjobt(int mt);
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

  // printf("mt=%d a1=%le b1=%le n1=%d s1=%le\n", mt, a1, b1, n1, s);

#pragma omp critical
  {
    sum += s;
    num++;

  }  // end critical

  return;
}

void parallel();

void parallel() {
  omp_set_num_threads(tp);
#pragma omp parallel
  {
    int mt = omp_get_thread_num();

    myjobt(tp * mp + mt);

  }  // end parallel

  while (num < mp);
}

int main(int argc, char *argv[]) {
  sscanf(argv[argc - 1], "%d", &tp);
  MyNetInit(&argc, &argv, &np, &mp, &nl, pname, &tick);
  if (np < 2) {
    t1 = MPI_Wtime();
    sum = integrate(f1, a, b, ni);
    t2 = MPI_Wtime();
    t3 = t2;
  } else {
    int i;
    double p;
    t1 = MPI_Wtime();
    parallel();
    t2 = MPI_Wtime();
    if (mp == 0) {
      MPI_Recv(&p, 1, MPI_DOUBLE, 1, MY_TAG, MPI_COMM_WORLD, &status);
      sum = sum + p;
      for (i = 2; i < np; i += 2) {
        MPI_Recv(&p, 1, MPI_DOUBLE, i, MY_TAG, MPI_COMM_WORLD, &status);
        sum = sum + p;
      }
    } else {
      if (mp % 2 == 0) {
        MPI_Recv(&p, 1, MPI_DOUBLE, mp + 1, MY_TAG, MPI_COMM_WORLD, &status);
        sum = sum + p;
        MPI_Send(&sum, 1, MPI_DOUBLE, 0, MY_TAG, MPI_COMM_WORLD);
      } else {
        MPI_Send(&sum, 1, MPI_DOUBLE, mp - 1, MY_TAG, MPI_COMM_WORLD);
      }
    }
    MPI_Barrier(MPI_COMM_WORLD);
    t3 = MPI_Wtime();
  }
  t1 = t2 - t1;
  t2 = t3 - t2;
  t3 = t1 + t2;
  fprintf(stderr, "mp=%d t1=%lf t2=%lf t3=%lf int=%22.15le\n", mp, t1, t2, t3,
          sum);
  MPI_Finalize();
  return 0;
}
```