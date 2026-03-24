# Linux tracing tools

Some occasionally useful systemtap and bpftrace scripts.

```sh
sudo stap -v -x `pgrep softIoc` futexs.stp
```

```sh
sudo stap -x `pgrep softIoc` --ldd alloc_time.stp
```

Details

- [Identifying Mutex Contention in EPICS IOCs](ioc_mutex_contention.md)
