# Zombie Db Connections in DB Pool 

### What happened 
is that all the TCP (Transmission Control Protocol) connections of the pool got into a sort of “zombie” state: broken yet not closed. 
The reason for that is the postgreSQL driver does not set a TCP timeout on the connections. The pool never attempted to close the connections 
and create new ones, yet no data would come in or out of these connections. 

### Why 
The reason that HikariCP is powerless to recover connections that are out of the pool is due to unacknowledged TCP traffic. TCP is a synchronous 
communication scheme that requires “handshaking” from both sides of the connection as packets are exchanged (SYN and ACK packets).
When TCP communication is abruptly interrupted, the client or server can be left awaiting the acknowledgement of a packet that will never come. 
The connection is therefore “stuck”, until an operating system level TCP timeout occurs. This can be as long as several hours, depending on the 
operating system TCP stack tuning.

From the point of view of the connection pool, all the connections were fine but for the newly promoted database there were no incoming connections
from the application. It took the application six hours to eventually fail, with new servers coming online the database access was restored 
and everything went back to normal. This is far from ideal.

### Solution
While rapid recovery isn’t a new concept I hadn’t heard of it. I only discovered it while investigating our incident. It turns out Hikari has a 
[brilliant page dedicated to it](https://github.com/brettwooldridge/HikariCP/wiki/Rapid-Recovery).Their advice is two fold:

- Ensure you don’t cache DNS for too long. The JVM default is indefinite for security reasons, but this didn’t age well with the advent of cloud computing. We changed the default such that any Guardian server with a JVM should only cache DNS for 60s. You can see how Max and Roberto implemented that change for our whole department here.
- Ensure you set a TCP timeout. Some drivers support it, some don’t. PostgreSQL supports it but sets it to unlimited by default. So by default connections that have been dropped might never be closed.

### Reference
[Faster PostgresSQL Failover Recovery](https://www.theguardian.com/info/2019/dec/02/faster-postgresql-connection-recovery)
