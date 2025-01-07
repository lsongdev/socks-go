# socks-go

simple socks5 client and server implementation in go

## example

### simple socks5 client

```go
func main(){

  client := socks.NewClient(&socks.ClientConfig{
    Host: "example.com",
    Port: 1080,
    User: &socks.User{
      Username: "username",
      Password: "password",
    },
  })
  conn, err := client.Dial("tcp", "example.com:80")
  if err != nil {
    log.Fatal(err)
  }
  defer conn.Close()
  if _, err := conn.Write([]byte("GET / HTTP/1.0\r\n\r\n")); err != nil {
    log.Fatal(err)
  }
  io.Copy(os.Stdout, conn)
}
```

### simple socks5 server

```go
package main

import (
  "io"
  "log"

  "github.com/lsongdev/socks-go/socks"
)

type MyHandler struct {
  socks.DefaultServerHandler

  client *socks.Client
}

func (h *MyHandler) HandleAuth(user *socks.User) error {
  log.Println("User", user)
  return nil
}

func (h *MyHandler) HandleRequest(conn *socks.PacketConn) {
  log.Println("-->", conn.Address())
  dest, err := h.client.Dial("tcp", conn.Address())
  if err != nil {
    log.Printf("Failed to connect to %s: %v", conn.Address(), err)
    conn.SendStatus(socks.HOST_UNREACHABLE) // Host unreachable
    return
  }
  defer dest.Close()
  // Send success response
  if err := conn.SendStatus(socks.SUCCESS); err != nil {
    log.Printf("Failed to send response: %v", err)
    return
  }
  // Start proxying data
  go func() {
    defer conn.Close()
    defer dest.Close()
    io.Copy(dest, conn)
  }()
  io.Copy(conn, dest)
}

func main() {

  client := socks.NewClient(&socks.ClientConfig{
    Host: "example.com",
    Port: 1080,
    User: &socks.User{
      Username: "username",
      Password: "password",
    },
  })

  h := &MyHandler{
    client: client,
  }
  server := socks.NewServer()
  log.Printf("Starting SOCKS5 server on :1080")
  if err := server.ListenAndServe(":1080", h); err != nil {
    log.Fatal(err)
  }
}
```

## license

see [LICENSE](https://github.com/lsongdev/.github/blob/main/LICENSE.md)