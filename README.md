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
        - This one change improved time from 1.25s to 20ms
    - If we run profile again we don't see our function, all we see is background noise and mallocgc
        - Which might be the difference between our program and wc.
        - It tells to run different type of profile. So let's switch to memory profiling.
        ```go
        defer profile.Start(profile.MemProfile, profile.ProfilePath(".")).Stop()
        ```
        - It catches the stacktrace that lead to that allocation.
        - We can see that time went down significantly as different types of profiling has different overhead.
        - We can see that almost all the allocations are coming from main readbyte
        - Click on view/source and we can see the line.
        - Note: We are sampling one out of every 1000 allocations. We can at the cost of some performance crank that up and say I want to profile every allocation, as below:
        ```go
        defer profile.Start(profile.MemProfile, profile.MemProfileRate(1), profile.ProfilePath(".")).Stop()
        ```
        - Now we can see that main readbyte is 1.2MB. Which means that memory usage is proportional to the data we feed it.
        - So why is it leaking?
            - It's leaking because we declare an array ```var buf [1]byte```, we slice that array into slice ```_, err := r.Read(buf[:])```, and we pass it to r.Read. Now what's the type of r? It's an io.Reader interface type. So the runtime and compiler don't know what read is actually going to be passed, could be file could be buffer or something else. We don't know what they are going to do with the content of the buffer we passed them. They might capture the address of one of the elements in the buffer and keep it. This forces this allocation to escape the heap and that's where all the memory allocation being reported. 