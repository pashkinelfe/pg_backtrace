pg_backtrace is a PostgreSQL extension to dump stack trace into the
logfile when no debugger is available.

Backtraces for SIGSEGV, SIGBUS, SIGINT and SIGFPE signals, and for
errorlevel greater or equal to ```pg_backtrace.level```
are dumped into logfile.

# Setup

```
make USE_PGXS=1 clean && make USE_PGXS=1 -j4 && make USE_PGXS=1 install
```

```sql
CREATE EXTENSION pg_backtrace;
SELECT pg_backtrace_init();
```

# Usage:

Stack traces are written into the logfile (even if dumping cores is disabled):

```
2018-11-12 18:25:52.636 MSK [24518] LOG:  Caught signal 11
2018-11-12 18:25:52.636 MSK [24518] CONTEXT:  	/home/username/postgresql/dist/lib/pg_backtrace.so(+0xe37) [0x7f6358838e37]
		/lib/x86_64-linux-gnu/libpthread.so.0(+0x11390) [0x7f63624e3390]
		/home/username/postgresql/dist/lib/pg_backtrace.so(pg_backtrace_force_crash+0) [0x7f6358838fb0]
		postgres: username postgres [local] SELECT() [0x5fe474]
		postgres: username postgres [local] SELECT() [0x6266a8]
		postgres: username postgres [local] SELECT(standard_ExecutorRun+0x15a) [0x60193a]
		postgres: username postgres [local] SELECT() [0x74168c]
		postgres: username postgres [local] SELECT(PortalRun+0x29e) [0x742a7e]
		postgres: username postgres [local] SELECT() [0x73e922]
		postgres: username postgres [local] SELECT(PostgresMain+0x1189) [0x73fde9]
		postgres: username postgres [local] SELECT() [0x47d5e0]
		postgres: username postgres [local] SELECT(PostmasterMain+0xd28) [0x6d0448]
		postgres: username postgres [local] SELECT(main+0x421) [0x47e511]
		/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0) [0x7f6361a13830]
		postgres: username postgres [local] SELECT(_start+0x29) [0x47e589]
```

Strictly debug only!!! To manually crash with SIGTRAP (where compatible) or SIGSEGV (otherwise) signal run:
```
SELECT pg_backtrace_force_crash();
```

### Set minimum error level for stack trace:

Default error level to be backtraced is ```fatal```. So all exceptions from ```fatal``` to higher error levels (```panic```) and signals SIGSEGV, SIGBUS, SIGINT and SIGFPE will print backtrace information which is dumped both in logfile and delivered to the client:

Valid values are: ```panic, fatal, error, warning, notice, info, log, debug, debug1, debug2, debug3, debug4, debug5```
It's not recommended to set anything below ERROR do avoid logs congestion. In fact useful values are only ```fatal``` and ```panic``` 

```
SET pg_backtrace.level = error;
```

See backtrace in the logfile and psql prompt:
```
postgres=# select count(*)/0.0 from pg_class;
ERROR:  division by zero
CONTEXT:  	postgres: username postgres [local] SELECT(numeric_div+0xbc) [0x7c5ebc]
	postgres: username postgres [local] SELECT() [0x5fe4e2]
	postgres: username postgres [local] SELECT() [0x610730]
	postgres: username postgres [local] SELECT() [0x6115ca]
	postgres: username postgres [local] SELECT(standard_ExecutorRun+0x15a) [0x60193a]
	postgres: username postgres [local] SELECT() [0x74168c]
	postgres: username postgres [local] SELECT(PortalRun+0x29e) [0x742a7e]
	postgres: username postgres [local] SELECT() [0x73e922]
	postgres: username postgres [local] SELECT(PostgresMain+0x1189) [0x73fde9]
	postgres: username postgres [local] SELECT() [0x47d5e0]
	postgres: username postgres [local] SELECT(PostmasterMain+0xd28) [0x6d0448]
	postgres: username postgres [local] SELECT(main+0x421) [0x47e511]
	/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0) [0x7f6361a13830]
	postgres: username postgres [local] SELECT(_start+0x29) [0x47e589]
```

### Determine the current state of stuck backend

If some backend is stuck with long running query or by other reason you do not know where it spends most of time:

- Use ```htop``` utility to find <pid> of the backend that is stuck based on CPU utilization and time spent: 

![alt text](https://github.com/pashkinelfe/pg_backtrace/blob/master/htop.jpg?raw=true)

- Send SIGINT signalto backend:
```
kill -2 <pid>
```

- See backtrace in the logfile:
```
2018-11-12 18:24:12.222 MSK [24457] LOG:  Caught signal 2
2018-11-12 18:24:12.222 MSK [24457] CONTEXT:  	/lib/x86_64-linux-gnu/libpthread.so.0(+0x11390) [0x7f63624e3390]
		/lib/x86_64-linux-gnu/libc.so.6(epoll_wait+0x13) [0x7f6361afa9f3]
		postgres: username postgres [local] SELECT(WaitEventSetWait+0xbe) [0x71e4de]
		postgres: username postgres [local] SELECT(WaitLatchOrSocket+0x8b) [0x71e93b]
		postgres: username postgres [local] SELECT(pg_sleep+0x98) [0x7babd8]
		postgres: username postgres [local] SELECT() [0x5fe4e2]
		postgres: username postgres [local] SELECT() [0x6266a8]
		postgres: username postgres [local] SELECT(standard_ExecutorRun+0x15a) [0x60193a]
		postgres: username postgres [local] SELECT() [0x74168c]
		postgres: username postgres [local] SELECT(PortalRun+0x29e) [0x742a7e]
		postgres: username postgres [local] SELECT() [0x73e922]
		postgres: username postgres [local] SELECT(PostgresMain+0x1189) [0x73fde9]
		postgres: username postgres [local] SELECT() [0x47d5e0]
		postgres: username postgres [local] SELECT(PostmasterMain+0xd28) [0x6d0448]
		postgres: username postgres [local] SELECT(main+0x421) [0x47e511]
		/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0) [0x7f6361a13830]
		postgres: username postgres [local] SELECT(_start+0x29) [0x47e589]
```

### Limitations

- When Postgres extension is loaded and initialized it is necessary to call
pg_backtrace_init() function once to be able to use this extension. This
function actually does nothing but _PG_init() registers signal handlers
for SIGSEGV, SIGBUS and SIGINT and executor run hook which setups exception
context.

- This extension is using backtrace function which is available at most Unixes.
But as it was mentioned in backtrace documentation:

    The symbol names may be unavailable without the use of special linker options.
	For systems using the GNU linker, it is necessary to use the -rdynamic
    linker option.  Note that names of "static" functions are not exposed,
	and won't be available in the backtrace.

Postgres is built without -rdynamic option. This is why not all function addresses
in the stack trace above are resolved. It is possible to use GDB (at development
host with correspondent postgres binaries) or Linux addr2line utility to
get resolve function addresses:

```
    $ addr2line -e ~/postgresql/dist/bin/postgres -a 0x5fe4e2
    0x00000000005fe4e2
    execExprInterp.c:?
```

Adapted by Pavel Borisov <pashkin.elfe@gmail.com> from https://github.com/postgrespro/pg_backtrace by Konstantin Knizhnik

