# Steps
- To profile your code add below line in the source
```go
defer profile.Start(profile.CPUProfile, profile.ProfilePath(".")).Stop()
```

- Build the project
```sh
go build pkg/examples/words/main.go
```

- Run the binary and id will create cpu.pprof file
```sh
time ./main moby.txt
```

- Look at the profile using pprof tool
```sh
go tool pprof -http=:8088 cpu.pprof
```

- If we look at the time. Most of the time was taken in syscall in the end.
    - If we run wc command for word count it almost does it instantaneously. So it's not syscall which is taking this much time.
    - We are probably making too many syscall. 
        - In Go all our read and write are unbuffered by default.
        - To fix it we buffer it