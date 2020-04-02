# go-mysql

### Example

```go
import (
    "github.com/gocuntian/go-mysql/replication"
    "os"
)
// Create a binlog syncer with a unique server id, the server id must be different from other MySQL's. 
// flavor is mysql or mariadb
cfg := replication.BinlogSyncerConfig {
    ServerID: 100,
    Flavor:   "mysql",
    Host:     "127.0.0.1",
    Port:     3306,
    User:     "root",
    Password: "",
}
syncer := replication.NewBinlogSyncer(cfg)

// Start sync with specified binlog file and position
streamer, _ := syncer.StartSync(mysql.Position{binlogFile, binlogPos})

// or you can start a gtid replication like
// streamer, _ := syncer.StartSyncGTID(gtidSet)
// the mysql GTID set likes this "de278ad0-2106-11e4-9f8e-6edd0ca20947:1-2"
// the mariadb GTID set likes this "0-1-100"

for {
    ev, _ := streamer.GetEvent(context.Background())
    // Dump event
    ev.Dump(os.Stdout)
}

// or we can use a timeout context
for {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    ev, err := s.GetEvent(ctx)
    cancel()

    if err == context.DeadlineExceeded {
        // meet timeout
        continue
    }

    ev.Dump(os.Stdout)
}
```

The output looks:

```
=== RotateEvent ===
Date: 1970-01-01 08:00:00
Log position: 0
Event size: 43
Position: 4
Next log name: mysql.000002

=== FormatDescriptionEvent ===
Date: 2014-12-18 16:36:09
Log position: 120
Event size: 116
Version: 4
Server version: 5.6.19-log
Create date: 2014-12-18 16:36:09

=== QueryEvent ===
Date: 2014-12-18 16:38:24
Log position: 259
Event size: 139
Salve proxy ID: 1
Execution time: 0
Error code: 0
Schema: test
Query: DROP TABLE IF EXISTS `test_replication` /* generated by server */
```

## Canal 
```go
cfg := NewDefaultConfig()
cfg.Addr = "127.0.0.1:3306"
cfg.User = "root"
// We only care table canal_test in test db
cfg.Dump.TableDB = "test"
cfg.Dump.Tables = []string{"canal_test"}

c, err := NewCanal(cfg)

type MyEventHandler struct {
    DummyEventHandler
}

func (h *MyEventHandler) OnRow(e *RowsEvent) error {
    log.Infof("%s %v\n", e.Action, e.Rows)
    return nil
}

func (h *MyEventHandler) String() string {
    return "MyEventHandler"
}

// Register a handler to handle RowsEvent
c.SetEventHandler(&MyEventHandler{})

// Start canal
c.Run()
```

## Client

### Example

```go
import (
    "github.com/gocuntian/go-mysql/client"
)

// Connect MySQL at 127.0.0.1:3306, with user root, an empty password and database test
conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test")

// Or to use SSL/TLS connection if MySQL server supports TLS
//conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test", func(c *Conn) {c.UseSSL(true)})

// or to set your own client-side certificates for identity verification for security
//tlsConfig := NewClientTLSConfig(caPem, certPem, keyPem, false, "your-server-name")
//conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test", func(c *Conn) {c.SetTLSConfig(tlsConfig)})

conn.Ping()

// Insert
r, _ := conn.Execute(`insert into table (id, name) values (1, "abc")`)

// Get last insert id
println(r.InsertId)

// Select
r, _ := conn.Execute(`select id, name from table where id = 1`)

// Handle resultset
v, _ := r.GetInt(0, 0)
v, _ = r.GetIntByName(0, "id") 
```

Tested MySQL versions for the client include:
- 5.5.x
- 5.6.x
- 5.7.x
- 8.0.x

## Server

### Example

```go
import (
    "github.com/gocuntian/go-mysql/server"
    "net"
)

l, _ := net.Listen("tcp", "127.0.0.1:4000")

c, _ := l.Accept()

conn, _ := server.NewConn(c, "root", "", server.EmptyHandler{})

for {
    conn.HandleCommand()
}
``` 

Another shell

```
mysql -h127.0.0.1 -P4000 -uroot -p 
//Becuase empty handler does nothing, so here the MySQL client can only connect the proxy server. :-) 
```

> ```NewConn()``` will use default server configurations:
> 1. automatically generate default server certificates and enable TLS/SSL support.
> 2. support three mainstream authentication methods **'mysql_native_password'**, **'caching_sha2_password'**, and **'sha256_password'**
>    and use **'mysql_native_password'** as default.
> 3. use an in-memory user credential provider to store user and password.
>
> To customize server configurations, use ```NewServer()``` and create connection via ```NewCustomizedConn()```.



## Driver


```go
package main

import (
    "database/sql"

    _ "github.com/gocuntian/go-mysql/driver"
)

func main() {
    // dsn format: "user:password@addr?dbname"
    dsn := "root@127.0.0.1:3306?test"
    db, _ := sql.Open(dsn)
    db.Close()
}
```

