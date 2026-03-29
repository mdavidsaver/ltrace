# Identifying Mutex Contention in EPICS IOCs

A note on how to use [`futexs.stp`](futexs.stp) to identify threads
and mutex participating in lock contention
in an [EPICS](https://epics-controls.org) IOC.
Uses [systemtap](http://sourceware.org/systemtap/) on Linux.

```sh
git clone https://github.com/epics-base/ltrace
```

It is typically necessary to install a compiler and Linux kernel debug information in order to use systemtap.
eg. on Debian

```sh
sudo apt install systemtap linux-image-amd64-dbg
```

For Debian 13 with Linux 6.13, it is currently (March 2026)
[necessary](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1131621)
to manually build/install system tap 5.3.

## epicsMutex on Linux

On Linux, an `epicsMutexID` wraps a `pthread_mutex_t`.
With glibc, the address of this `pthread_mutex_t` is used as the `uaddr`
for futex system calls into the Linux kernel.

The pthread mutex implementation has 1) a fast path which is entirely in
userspace when a mutex is not locked.
In this case, no syscall is made.
If the mutex is locked a `futex(..., FUTEX_WAIT, ...)` syscall is made.
By the time the futex call is evaluated, it is possible that 2) the mutex has become unlocked,
in which case the futex call takes the lock and returns without a context switch.
Finally, if the lock is still held, 3) the futex call will block the calling thread until
the mutex is unlocked.

In the output of the `futexs.stp` script these cases are:

1. Invisible
2. `Contention without context switch` table
3. `Contention with context switch` table

## Run target process

Assumes analysis for some period of time after IOC start.

Start your IOC process as normal.
Take note of the PID (as `$IOCPID`).

## Run systemtap

```sh
$ sudo stap -v -x $IOCPID futexs.stp
Pass 1: parsed user script ...
Start with target=$IOCPID...
```

When the "Start ..." message is seen, the target process is being monitored.
If necessary, take action to induce load/contention.
When finished, interrupt with `Ctrl+c`.
A report will be printed like the following.

```
Contention with context switch
[320795] lock 0x7fdd6c001070 contended 37 times, 4 avg 190688 max us
[320795] lock 0x7fdd6c007e90 contended 1 times, 11 avg 80 max us
[320799] lock 0x55d45d8e70a0 contended 9 times, 9 avg 17061 max us
[320800] lock 0x55d45d8e7550 contended 10 times, 30 avg 218 max us
[320803] lock 0x55d45d8e60d0 contended 6 times, 5 avg 571 max us
[320803] lock 0x55d45d9586d0 contended 366 times, 67 avg 35734 max us
[320810] lock 0x55d45d9586d0 contended 1 times, 39 avg 1000013 max us
[320817] lock 0x7fdd6c001070 contended 64 times, 4 avg 61569 max us
[320817] lock 0x55d45d840810 contended 26 times, 3 avg 25971 max us
[320817] lock 0x55d45d985070 contended 3 times, 13 avg 451 max us
[320817] lock 0x7fdd28000030 contended 1 times, 5 avg 8753 max us

Contention without context switch
[320795] lock 0x7fdd6c007e90 contended 2 times, 11 avg 18 max us
[320795] lock 0x7fdd6c001070 contended 206 times, 4 avg 18 max us
[320795] lock 0x7fdd28000030 contended 2 times, 3 avg 4 max us
[320795] lock 0x55d45d840810 contended 7 times, 9 avg 27 max us
[320799] lock 0x55d45d8e70a0 contended 37 times, 9 avg 22 max us
[320800] lock 0x55d45d8e7550 contended 1 times, 30 avg 30 max us
[320803] lock 0x55d45d9586d0 contended 8879 times, 67 avg 383 max us
[320803] lock 0x55d45d8e60d0 contended 2 times, 5 avg 7 max us
[320810] lock 0x55d45d9586d0 contended 2 times, 39 avg 54 max us
[320817] lock 0x7fdd6c001070 contended 100 times, 4 avg 31 max us
[320817] lock 0x55d45d840810 contended 36 times, 3 avg 16 max us
[320817] lock 0x55d45d985070 contended 1 times, 13 avg 13 max us
[320817] lock 0x7fdd28000030 contended 2 times, 5 avg 7 max us
[320817] lock 0x55d45d985040 contended 1 times, 3 avg 3 max us
```

Here the first column (eg. `[324383]`) is a Linux thread ID.
The third column (eg `0x55a72157a2a0`) is the address of the futex.

To make sense of these numbers, run the following commands at the IOC shell.

```sh
epicsThreadShowAll
epicsMutexShowAll 0 1
dblsr '' 1
```

## epicsThreadShowAll

The `LWP ID` shows the Linux thread ID for those threads created through the epicsThread API.
eg. `[320803]` is the `dbCaLink` thread.

```
epics> epicsThreadShowAll 
            NAME       EPICS ID   LWP ID   OSIPRI  OSSPRI  STATE
          _main_ 0x55d45d81b520   320791      0       0       OK
          errlog 0x55d45d8235e0   320793     10       0       OK
          PVXTCP 0x55d45d840970   320795     18       0       OK
     IfMapDaemon 0x55d45d82a520   320796      0       0       OK
          PVXUDP 0x55d45d8df390   320797     16       0       OK
          taskwd 0x55d45d8e6b90   320798     10       0       OK
      timerQueue 0x55d45d8e72b0   320799     70       0       OK
           cbLow 0x55d45d8e75c0   320800     59       0       OK
        cbMedium 0x55d45d8e5a20   320801     64       0       OK
          cbHigh 0x55d45d8e5d90   320802     71       0       OK
        dbCaLink 0x55d45d8e61b0   320803     50       0       OK
         pvxlink 0x55d45d9560f0   320804     50       0       OK
         PVXCTCP 0x55d45d956820   320805     20       0       OK
        scanOnce 0x55d45d97ac50   320806     67       0       OK
         scan-10 0x55d45d97af20   320807     60       0       OK
          scan-5 0x55d45d97b1c0   320808     61       0       OK
          scan-2 0x55d45d97b460   320809     62       0       OK
          scan-1 0x55d45d97b700   320810     63       0       OK
        scan-0.5 0x55d45d97b9a0   320811     64       0       OK
        scan-0.2 0x55d45d97bc40   320812     65       0       OK
        scan-0.1 0x55d45d97bee0   320813     66       0       OK
         CAS-TCP 0x55d45d984840   320814     18       0       OK
         CAS-UDP 0x55d45d984ae0   320815     16       0       OK
      CAS-beacon 0x55d45d984d80   320816     17       0       OK
      qsrvSingle 0x55d45d987240   320817     19       0       OK
       qsrvGroup 0x55d45d987760   320818     19       0       OK
```0x55d45d81b670

## epicsMutexShowAll

The futex addresses (`uaddr=0x...`) for mutex allocated through the epicsMutex API are shown by `epicsMutexShowAll`.
eg. `uaddr=0x55d45d9586d0` is associated with an epicsMutex `0x55d45d9586b0` allocated at `dbLock.c:86`,
which in this instance is associated with a database record lockset,
for which we refer to the `dblsr` output.

```
epics> epicsMutexShowAll 0 1
ellCount(&mutexList) 319
PI is enabled
...
epicsMutexId 0x55d45d9586b0 source ../db/dbLock.c line 86
    pthread_mutex_t* uaddr=0x55d45d9586d0
...
epicsMutexId 0x55d45d81b650 source ../osi/epicsGeneralTime.c line 91
    pthread_mutex_t* uaddr=0x55d45d81b670
epicsMutexId 0x55d45d81b6a0 source ../osi/epicsGeneralTime.c line 94
    pthread_mutex_t* uaddr=0x55d45d81b6c0
epicsMutexId 0x55d45d81bb90 source ../iocsh/iocsh.cpp line 1597
    pthread_mutex_t* uaddr=0x55d45d81bbb0
epicsMutexId 0x55d45d81dc50 source ../gpHash/gpHashLib.c line 57
    pthread_mutex_t* uaddr=0x55d45d81dc70
epicsMutexId 0x55d45d821130 source ../iocsh/initHooks.c line 41
    pthread_mutex_t* uaddr=0x55d45d821150
... omitting a tremendous number of lines
```

## dblsr

The `dblsr` (DataBase Lock Set Report) function shows the current association between epicsMutex and the list of records.

```
epics> dblsr '' 1
Lock Set 3 3 members 3 refs epicsMutexId 0x55d45d9586b0
cnt
measure
drive
```

## Putting it all together

In this example, the following database was loaded, with `ODLY=0.001`
to setup a `<= 1 kHz` counter ("cnt"),
which is sampled at 1Hz to compute a "measure"d loop rate.

Since all record reference each other through some DB link,
all are part of the same lockset.

```
record(calc, "measure") {
    field(SCAN, "1 second")
    field(INPA, "drive NPP")
    field(CALC, "C:=A-B;B:=A;C")
    field(PREC, "2")
    field(EGU , "Hz")
}

record(calcout, "drive") {
    field(PINI, "RUNNING")
    field(INPA, "drive")
    field(CALC, "A+1")
    field(OUT , "drive.PROC CA")
    field(ODLY, "$(ODLY=1.0)") # set to 0.001 for fun!
    field(FLNK, "cnt")
}

record(calc, "cnt") {
    field(INPA, "cnt")
    field(CALC, "A+1")
    field(TSEL, "drive.TIME")
}
```

Looking back at a subset of the earlier systemtap output for the most contended lock `0x55d45d9586d0`.

```
Contention with context switch
[320803] lock 0x55d45d9586d0 contended 366 times, 67 avg 35734 max us
[320810] lock 0x55d45d9586d0 contended 1 times, 39 avg 1000013 max us

Contention without context switch
[320803] lock 0x55d45d9586d0 contended 8879 times, 67 avg 383 max us
[320810] lock 0x55d45d9586d0 contended 2 times, 39 avg 54 max us
```

Referring to the associations show be the IOC shell commands,
we can make the following substitutions.

```
TID 320803 -> dbCaLink
TID 320810 -> scan-1

lock uaddr=0x55d45d9586d0 -> epicsMutexId 0x55d45d9586b0 -> Lock Set ...
```


```
Contention with context switch
[dbCaLink] Lock Set 3 contended 366 times, 67 avg 35734 max us
[scan-1] Lock Set 3 contended 1 times, 39 avg 1000013 max us

Contention without context switch
[dbCaLink] Lock Set 3 contended 8879 times, 67 avg 383 max us
[scan-1] Lock Set 3 contended 2 times, 39 avg 54 max us
```

So we can see that the contention is between the CA link worker thread (`dbCaLink`)
and the `1 second` periodic scan thread (`scan-1`).

```
record(calc, "measure") {
    field(SCAN, "1 second")
    ...
}

record(calcout, "drive") {
    field(PINI, "RUNNING")
    field(OUT , "drive.PROC CA") # re-queues itself
    ...
}
```

The story we can tell from this is that the fast loop effected
when the "drive" record re-queues itself occasionally has to contend
while waiting as the "measure" record samples the counter.

The reverse will also happen, when "measure" has to wait for "drive".
Although about 0.1% of the time.
