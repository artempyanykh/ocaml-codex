# Fuzz Test Patterns

Use these Crowbar patterns after the main fuzz skill has established the project
layout and registration style.

## Crash-Safety

Parsers must not crash on arbitrary input.

```ocaml
open Crowbar
open Fuzz_common

(** Decode - must not crash on arbitrary input. *)
let test_decode buf =
  let buf = truncate buf in
  let _ = Foo.decode (to_bytes buf) in
  ()

(** Decode with exceptions - must not crash. *)
let test_decode_exn buf =
  let buf = truncate buf in
  (try ignore (Foo.decode_exn (to_bytes buf)) with _ -> ());
  ()

let run () =
  add_test ~name:"foo: decode crash safety" [ bytes ] test_decode;
  add_test ~name:"foo: decode_exn crash safety" [ bytes ] test_decode_exn
```

Use `bytes` for arbitrary binary input. Use `ignore` to discard results without
warnings.

## Encode/Decode Roundtrip

Valid decoded values must round-trip through the encoder.

```ocaml
(** Roundtrip - valid values must round-trip. *)
let test_roundtrip buf =
  let buf = truncate buf in
  match Foo.decode (to_bytes buf) with
  | Error _ -> ()
  | Ok original ->
      let encoded = Foo.encode original in
      match Foo.decode encoded with
      | Error _ -> fail "re-decode failed"
      | Ok decoded ->
          if original <> decoded then fail "roundtrip mismatch"

let run () =
  add_test ~name:"foo: roundtrip" [ bytes ] test_roundtrip
```

If the initial decode fails, the input was invalid and the test should pass. If
re-decoding an encoded value fails, that is a bug.

## Constrained Type Roundtrip

```ocaml
(** APID roundtrip - valid values must round-trip. *)
let test_apid_roundtrip n =
  match Apid.of_int n with
  | None -> if n >= 0 && n <= 2047 then fail "should accept valid value"
  | Some apid ->
      let n' = Apid.to_int apid in
      if n <> n' then fail "roundtrip mismatch"

let run () =
  add_test ~name:"apid: roundtrip" [ range 2048 ] test_apid_roundtrip
```

## Boundary Tests

```ocaml
(** Max valid value. *)
let test_max_valid () =
  match Apid.of_int 2047 with
  | None -> fail "2047 should be valid"
  | Some apid -> if Apid.to_int apid <> 2047 then fail "value mismatch"

(** Min valid value. *)
let test_min_valid () =
  match Apid.of_int 0 with
  | None -> fail "0 should be valid"
  | Some apid -> if Apid.to_int apid <> 0 then fail "value mismatch"

let run () =
  add_test ~name:"apid: max_valid" [ const () ] test_max_valid;
  add_test ~name:"apid: min_valid" [ const () ] test_min_valid
```

Use `[ const () ]` for tests with no random input. Do not use `[]` as the
generator list.

## Invalid Input Rejection

```ocaml
(** Values above max must be rejected. *)
let test_invalid_above n =
  let invalid = 2048 + n in
  match Apid.of_int invalid with
  | None -> ()
  | Some _ -> fail "should reject values > 2047"

(** Negative values must be rejected. *)
let test_invalid_negative n =
  let invalid = -(n + 1) in
  match Apid.of_int invalid with
  | None -> ()
  | Some _ -> fail "should reject negative values"

let run () =
  add_test ~name:"apid: invalid_above" [ range 1000 ] test_invalid_above;
  add_test ~name:"apid: invalid_negative" [ range 1000 ] test_invalid_negative
```

## Pretty-Printer Safety

```ocaml
(** Pretty-print - must not crash. *)
let test_pp buf =
  let buf = truncate buf in
  match Foo.decode (to_bytes buf) with
  | Error _ -> ()
  | Ok v -> let _ = Format.asprintf "%a" Foo.pp v in ()

let run () =
  add_test ~name:"foo: pp" [ bytes ] test_pp
```

## State Machine Transitions

```ocaml
(** Test valid state transitions. *)
let test_activate_pending kid algo material_buf =
  let material = to_bytes material_buf in
  if Bytes.length material = 0 then ()
  else
    let key = Key.v ~kid ~algorithm:algo ~material in
    match Key.activate key with
    | Error _ -> ()
    | Ok active_key ->
        if Key.state active_key <> Key.Active then fail "wrong state"

(** Test invalid state transitions return errors. *)
let test_activate_empty_fails kid algo =
  let key = Key.empty ~kid ~algorithm:algo in
  match Key.activate key with
  | Ok _ -> fail "should fail on Empty key"
  | Error (Key.Invalid_state_transition _) -> ()
  | Error _ -> fail "wrong error type"

let run () =
  add_test ~name:"key: activate Pending" [ uint8; uint8; bytes ]
    test_activate_pending;
  add_test ~name:"key: activate Empty fails" [ uint8; uint8 ]
    test_activate_empty_fails
```

## Unit Conversion Roundtrips

```ocaml
(** Nanoseconds roundtrip. *)
let test_ns_roundtrip n =
  let d = Duration.of_ns n in
  let n' = Duration.to_ns d in
  if n <> n' then fail "ns roundtrip mismatch"

(** Microseconds to milliseconds conversion. *)
let test_us_to_ms n =
  let us = Int64.of_int n in
  let d = Duration.of_us us in
  let ms = Duration.to_ms d in
  let expected = Int64.div us 1000L in
  if ms <> expected then fail "us to ms conversion failed"

let run () =
  add_test ~name:"duration: ns_roundtrip" [ int64 ] test_ns_roundtrip;
  add_test ~name:"duration: us_to_ms" [ range 1000000 ] test_us_to_ms
```

## Filestore and Resource Operations

```ocaml
(** Test create/exists invariant. *)
let test_create_exists name_buf =
  let name = Bytes.to_string (to_bytes name_buf) in
  if String.length name = 0 then ()
  else
    let fs = Filestore.in_memory () in
    match Filestore.create fs name with
    | Error _ -> ()
    | Ok () ->
        if not (Filestore.exists fs name) then
          fail "created file should exist"

let run () =
  add_test ~name:"filestore: create_exists" [ bytes ] test_create_exists
```

## Generators Reference

| Generator | Type | Use for |
|-----------|------|---------|
| `bytes` | `string` | Arbitrary binary data |
| `uint8` | `int` | 0-255 |
| `int8` | `int` | -128 to 127 |
| `int32` | `int32` | Full int32 range |
| `int64` | `int64` | Full int64 range |
| `range n` | `int` | 0 to n-1 |
| `bool` | `bool` | true/false |
| `const v` | `'a` | Fixed value |
| `list gen` | `'a list` | Lists of generated values |
| `option gen` | `'a option` | Some/None |
