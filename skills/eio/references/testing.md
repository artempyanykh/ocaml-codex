# Eio Testing

Use this reference when testing Eio code with `eio.mock`. The library name is
`eio.mock`, not `eio_mock`.

## Test Setup Pattern

```ocaml
let setup_test f () =
  Mirage_crypto_rng_unix.use_default ();
  Eio_main.run @@ fun env ->
  Eio.Switch.run @@ fun sw ->
  let fs = Eio.Stdenv.fs env in
  let tmp = Eio.Path.(fs / Filename.get_temp_dir_name () / "test-dir") in
  (try Eio.Path.rmtree tmp with _ -> ());
  Eio.Path.mkdirs ~exists_ok:true ~perm:0o755 tmp;
  Fun.protect
    ~finally:(fun () -> try Eio.Path.rmtree tmp with _ -> ())
    (fun () -> f ~sw tmp)

let test_something = setup_test @@ fun ~sw tmp ->
  (* test code here *)
```

## Mock Flow

Always end mock reads with `` `Raise End_of_file``. Otherwise the test may hang.

```ocaml
let test_api_call () =
  Eio_main.run @@ fun _env ->
  let flow = Eio_mock.Flow.make "response" in
  Eio_mock.Flow.on_read flow [
    `Return "{\"ok\": true}";
    `Raise End_of_file;
  ];
  (* use flow *)
```

## Simulating Chunked Data

```ocaml
let test_partial_reads () =
  Eio_main.run @@ fun _env ->
  let flow = Eio_mock.Flow.make "chunked" in
  Eio_mock.Flow.on_read flow [
    `Return "\x00\x01";
    `Return "\x02\x03";
    `Raise End_of_file;
  ];
  (* Parser must handle values spanning chunks *)
```

## Mock Network

```ocaml
let test_network () =
  Eio_main.run @@ fun _env ->
  let net = Eio_mock.Net.make "mocknet" in
  let flow = Eio_mock.Flow.make "conn" in
  Eio_mock.Net.on_connect net [`Return flow];
  Eio_mock.Flow.on_read flow [`Return "data"; `Raise End_of_file];
  (* test network operations *)
```

## Deadlock Detection

`Eio_mock.Backend.run` automatically detects deadlocks in tests.

## Mock Testing Example

**Bad**: mock flow without `End_of_file` hangs forever

```ocaml
let test_read () =
  Eio_main.run @@ fun _env ->
  let flow = Eio_mock.Flow.make "test" in
  Eio_mock.Flow.on_read flow [
    `Return "data";
    (* Missing End_of_file! Test hangs *)
  ];
  read_all flow
```

**Good**: always end mock reads with `End_of_file`

```ocaml
let test_read () =
  Eio_main.run @@ fun _env ->
  let flow = Eio_mock.Flow.make "test" in
  Eio_mock.Flow.on_read flow [
    `Return "data";
    `Raise End_of_file;
  ];
  read_all flow
```
