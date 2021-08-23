# Context in Go

> [Source](https://www.sohamkamani.com/golang/context-cancellation-and-values/)

## When to use context

Usually when we deal with async operation for example:

1. HTTP request call.
2. Fetch data from database.

## What can context do

1. Send signal to terminate the operation.
2. Carry miscellaneous data.

### Context Cancellation in Go

1. Listening for the cancellation event
2. Emitting the cancellation event

#### Listening for the cancellation event

```
func main() {
	http.ListenAndServe(":8000", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		fmt.Fprint(os.Stdout, "processing request\n")
		select {
		case <-time.After(2 * time.Second):
			w.Write([]byte("request processed"))
		case <-ctx.Done():
			fmt.Fprint(os.Stderr, "request cancelled\n")
		}
	}))
}
```

Use the browser to open `localhost:8000` if close it before 2 second will see `request cancelled` on terminal.

#### Emitting a cancellation event

```

func operation1(ctx context.Context) error {
	time.Sleep(100 * time.Millisecond)
	return errors.New("failed")
}

func operation2(ctx context.Context) {
	select {
	case <-time.After(500 * time.Millisecond):
		fmt.Println("done")
	case <-ctx.Done():
		fmt.Println("halted operation2")
	}
}

func main() {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)

	go func() {
		err := operation1(ctx)
		if err != nil {
			cancel()
		}
	}()

	operation2(ctx)
}

```

Because of go routine when `operation1` return error it will cancel context as well, so we can see `halted operation2`
on terminal window.

#### Context timeout

```
// The context will be cancelled after 3 seconds
// If it needs to be cancelled earlier, the `cancel` function can
// be used, like before
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
```

#### Context deadline

```
// Setting a context deadline is similar to setting a timeout, except
// you specify a time when you want the context to cancel, rather than a duration.
// Here, the context will be cancelled on 2009-11-10 23:00:00
ctx, cancel := context.WithDeadline(ctx, time.Date(2009, time.November, 10, 23, 0, 0, 0, time.UTC))
```

#### Context timeout example

```
func main() {
	ctx := context.Background()
	ctx, _ = context.WithTimeout(ctx, 100*time.Millisecond)

	req, _ := http.NewRequest(http.MethodGet, "http://google.com", nil)

	req = req.WithContext(ctx)

	client := &http.Client{}
	res, err := client.Do(req)
	if err != nil {
		fmt.Println("Request failed:", err)
		return
	}
	fmt.Println("Response received, status code:", res.StatusCode)
}
```

Success

```
ctx, _ = context.WithTimeout(ctx, 200*time.Millisecond)
```

Fail

```
ctx, _ = context.WithTimeout(ctx, 100*time.Millisecond)
```

### Context values

Use `context.WithValue` to set and modify the value

```
ctx := context.WithValue(context.Background(), keyID, rand.Int())
ctx = context.WithValue(ctx, keyID, 14)
```

## Do not

1. Cancel context more than once.
2. When you need to receive context cancel signal do not wrap it with `WithTimeout` or `WithCancel`.