# grpcdynamic

Dynamic gRPC registration mechanism.

## Usage
`grpcdynamic` provides dynamic gRPC service registration with arbitrary implementations.

``` go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"github.com/ktr0731/grpcdynamic"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

type req struct {
	Name string
}

type res struct {
	Message string
}

func main() {
	service := grpcdynamic.NewService("api.Example")
	service.RegisterUnaryMethod("Unary", new(req), new(res), func(ctx context.Context, in interface{}) (interface{}, error) {
		req := in.(*req)
		return &res{Message: fmt.Sprintf("hi, %s", req.Name)}, nil
	})
	srv := grpcdynamic.NewServer([]*grpcdynamic.Service{service})
	reflection.Register(srv)

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	done := make(chan struct{})
	go func() {
		if err := startServer(ctx, srv); err != nil {
			log.Fatal(err)
		}
		close(done)
	}()

	res, err := callUnaryMethod(service.FullMethodName("Unary"))
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(res.Message)

	cancel()
	<-done
}

func startServer(ctx context.Context, srv *grpc.Server) error {
	l, err := net.Listen("tcp", ":50051")
	if err != nil {
		return err
	}

	go func() {
		<-ctx.Done()
		srv.Stop()
	}()

	if err := srv.Serve(l); err != nil {
		return err
	}
	return nil
}

func callUnaryMethod(name string) (*res, error) {
	conn, err := grpc.Dial(
		":50051",
		grpc.WithInsecure(),
		grpc.WithDefaultCallOptions(grpc.CallContentSubtype(grpcdynamic.CodecName)),
	)
	if err != nil {
		return nil, err
	}
	defer conn.Close()

	var res res
	if err := conn.Invoke(context.Background(), name, &req{Name: "foo"}, &res); err != nil {
		return nil, err
	}
	return &res, nil
}
```
