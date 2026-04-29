## Snippets

Helpful code snippets that stops me from opening 20 instances of VScode

### Graceful HTTP Shutdown
```go
func main() {
	srv := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	go func() {
		log.Println("Starting server...")
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatalf("Server could not start: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	<-quit

	log.Println("Shutting down server...")

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		log.Fatalf("Server could not shutdown: %v", err)
	}
}
```

### Parse Pagination Filters From Request
```go
type GetCategoryFilter struct {
	Limit  int
	Offset int
	Page   int
}

parseCategoryFilters(r *http.Request) (*GetCategoryFilter, error) {
	query := r.URL.Query()

	limit := 10
	page := 1

	if limitStr := query.Get("limit"); limitStr != "" {
		parsedLimit, err := strconv.Atoi(limitStr)
		if err != nil {
			return nil, fmt.Errorf("invalid 'limit' parameter")
		}
		limit = parsedLimit
	}

	if limit > 100 || limit < 1 {
		//reset limit to default when it exceeds this bound
		limit = 10
	}

	if pageStr := query.Get("page"); pageStr != "" {
		parsedPage, err := strconv.Atoi(pageStr)
		if err != nil {
			return nil, fmt.Errorf("invalid 'limit' parameter")
		}
		page = parsedPage
	}

	return &models.GetCategoryFilter{
		Limit:  limit,
		Page:   page,
		Offset: (page - 1) * limit,
	}, nil
}
```

### Networking

#### Concurrent TCP connection

```go
	l, err := net.Listen("tcp", "0.0.0.0:9092")
	if err != nil {
		fmt.Println("Failed to bind to port 9092")
		os.Exit(1)
	}

	for {
		conn, err := l.Accept()
        if err != nil {
			fmt.Println("Error accepting connection: ", err.Error())
			os.Exit(1)
		}

        go func(conn net.Conn) {
            defer conn.Close
        }(conn)
    }
```

#### Reading Conn

##### Reading Conn - Delimeter

This is to used for decoding data in formats like `RESP` or `HTTP\1.1`.
Examples:

- [RESP parser](https://github.com/knightfall22/redis-server-clone/blob/master/app/resp.go)

```go
        //+main.go
        go func(conn net.Conn) {
            defer conn.Close
            resp := NewResp(conn)

				value, err := resp.Read()
				if err != nil {
					fmt.Println("Error reading from connection", err.Error())
					return
				}

                continue
        }(conn)
```

```go
    //+parser.go
    type Resp struct {
        reader *bufio.Reader
    }

    func NewResp(rd io.Reader) *Resp {
        return &Resp{reader: bufio.NewReader(rd)}
    }
```

##### Reading Conn - Unclear Delimeter

In cases like the Kafka protocol use something like this. Keep in mind that it not advisable to read `Conn` to `EOF`.
I do not know why it bugs out but it does

```go
    func (p *Processor) read() ([]byte, error) {
        //read the 4 byte message size
        size := make([]byte, 4)
        _, err := io.ReadFull(p.rd, size)
        if err != nil && err != io.EOF {
            return nil, err
        }

        msgLength := int32(binary.BigEndian.Uint32(size))

        //Read the request body
        req := make([]byte, msgLength)
        _, err = io.ReadFull(p.rd, req)
        if err != nil && err != io.EOF {
            return nil, err
        }

        return req, nil
    }
```

#### Closing Conn

##### Close by signaling

View [here](https://github.com/knightfall22/raft/blob/master/raft/server.go#L75)

```go
        go func() {
		defer s.wg.Done()

		for {
			conn, err := s.listener.Accept()
			if err != nil {
				select {
				case <-s.quit:
					log.Printf("closing server connection")
					return
				default:
					log.Fatal("accept error:", err)
				}
			}
			s.wg.Add(1)
			go func() {
				defer s.wg.Done()
				s.rpcServer.ServeConn(conn)
			}()
		}
	}()
```

```go
    //+L116
    func (s *Server) Shutdown() {
        s.cm.Stop()
        close(s.quit)
        s.listener.Close()
        s.wg.Wait()
    }
```

### Binary

#### Reading and Writing

```go
//Reading
    var ID int32
	err := binary.Read(p, binary.BigEndian, &id)
```

```go
    //Writing
    id := 200
	err := binary.Write(p, binary.BigEndian, id)
```

#### Encoding

##### Zig Zag Encoding and Varint

Zig Zag Encoding is an encoding scheme that efficiently encode signed integers to unsigned so that negetive numbers don't tak up space when expressed as a varint.

```go
    func zigzagEncode(n int32)  {
        num := uint64(uint32(n<<1) ^ uint32(n>>31))

        tmp := make([]byte, binary.MaxVarintLen64)
        limit := binary.PutUvarint(tmp, num)

        var buf bytes.Buffer
        _, err := buf.Write(tmp[:limit])
        if err != nil {
            log.Fatalln(err)
        }

        eNum, err := binary.ReadVarint(&buf)
        if err != nil {
            log.Fatalln(err)
        }
        //Warning: eNum returns the zig-zag encoded value you'll still need to decode it
        //Also this fuction is more of an illustration
    }

    func zigzagEncode64(n int64)  {
        num := (n<<1) ^ (n>>63)

        tmp := make([]byte, binary.MaxVarintLen64)
        limit := binary.PutUvarint(tmp, num)

        var buf bytes.Buffer
        _, err := buf.Write(tmp[:limit])
        if err != nil {
            log.Fatalln(err)
        }

        eNum, err := binary.ReadVarint(&buf)
        if err != nil {
            log.Fatalln(err)
        }
        //Warning: eNum returns the zig-zag encoded value you'll still need to decode it
        //Also this fuction is more of an illustration
    }
```

###### Write and Read from Varint

Varints, short for variable-length integers, are a way of encoding integers using a variable number of bytes.
Each byte has 7 bits of data and 1 continuation bit. In Little-Endian order the most significant bit (MSB) of each byte (bit 7) is used to indicate whether more bytes follow
`1` indicate that theres more `0` indicates that there isn't.

For more information - [here](https://protobuf.dev/programming-guides/encoding/#signed-ints). 
Also, you can read the std library varint pkgs

```go
    func WriteUvarint(w *bytes.Buffer, x uint64) error {
        tmp := make([]byte, binary.MaxVarintLen64)
        limit := binary.PutUvarint(tmp, x)

        _, err := w.Write(tmp[:limit])

        return err

    }
```

```go
func ReadUVarint(b *bytes.Reader) (uint64, error) {
	uv, err := binary.ReadUvarint(b)

	return uv), err
}

```

##

#### Fetches path directory in relation to root

```go
	relDir, err := filepath.Rel(root, filepath.Dir(path))
	if err != nil {
		return err
	}
```

## Running programs with code

#### Opening default app for a file

You can open a file using `os/exec`. Snippet from [here](https://github.com/knightfall22/cli-book/blob/main/3_mdp/main.go#L124). This snippet runs start command on the cmd to open an html file

```go
	cName := ""
	cParams := []string{}

	// Define executable based on OS
	switch runtime.GOOS {
	case "linux":
		cName = "xdg-open"
	case "windows":
		cName = "cmd.exe"
		cParams = []string{"/C", "start"}
	case "darwin":
		cName = "open"
	default:
		return fmt.Errorf("OS not supported")
	}

	cParams = append(cParams, fname)

	// Locate executable in PATH
	cPath, err := exec.LookPath(cName)

    //Path C:\WINDOWS\system32\cmd.exe
    //Params [/C start C:\Users\ELITEG~1\AppData\Local\Temp\mdp3455502293.html]

	if err != nil {
		return err
	}

	if err := exec.Command(cPath, cParams...).Run(); err != nil {
		return err
	}
```

### MISC

- <random command> | tee result.txt - Saves result from STDOUT to result.txt
- `go get -u -v golang.org/x/tools/cmd/benchcmp` - Can be used to compare two benchmark results
- `go test -bench . -benchtime=10x -run ^$ -cpuprofile cpu01.pprof` - Generate benchmark pprof file
- `go test -bench . -benchtime=10x -run ^$ -memprofile mem00.pprof` - Generate memory allocation pprof file
- `go tool pprof -alloc_space mem00.pprof` - enter pprof interactive mode for mem allocation
