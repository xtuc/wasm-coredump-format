# Wasm Coredump Format

> In computing, a core dump, [...] consists of the recorded state of the working memory of a computer program at a specific time, generally when the program has crashed or otherwise terminated abnormally. In practice, other key pieces of program state are usually dumped at the same time, including the processor registers, which may include the program counter and stack pointer, memory management information, and other processor and operating system flags and information. A snapshot dump (or snap dump) is a memory dump requested by the computer operator or by the running program, after which the program is able to continue. Core dumps are often used to assist in diagnosing and debugging errors in computer programs.

-- <cite>[Core dump on Wikipedia]</cite>

## Motivations

The motivations are the same than usual post-mortem debugging. The main target environment is serverless and client-side environments (like browsers) are not considered.

## Idea

When WebAssembly enters a trap, it starts unwinding and collects debugging information. For each stack frame the state of the WebAssembly instance can be captured by collecting the values in locals (includes function parameters), globals and on the stack, along with an offset in the code section. Values are written into the `coredump` struct.

The post mortem analyze is done using the `coredump` struct, the WebAssembly linear memory (also called process image) and [DWARF] informations. Similar to the debugging flow with [gdb].

### Runtime support

For experimenting, runtime support is not strictly necessary. Tools can rewrite the Wasm binary to inject code that will unwinding the stack, collect debugging information and write the `coredump` struct, for instance [wasm-edit coredump]. Such a transformation has important limitations. For simplicty, [wasm-edit] places the `coredump` at offset 0 of the linear memory, however the proper placement hasn't been specified yet.

Example to demonstrate how to the coredump generation works with [wasm-edit]:
```js
const { instance } = await WebAssembly.instantiateStreaming(...);

try {
    instance.exports.main();
} catch (err) {
    const image = instance.memory();
    someStorage.put("coredump.1234", image);
}
```

### Security and privacy considerations

Using the WebAssembly linear memory for debugging exposes the risk of seeing, manipulating and/or collecting sensitive informations.
Handling of sensitive data is out of scope.

No particular security considerations.

### Debugger support

[gdb] doesn't support Wasm coredump and it's unclear if it can.
Wasm coredump differ from ELF coredump in a few significant ways:
- Wasm semantics; usage of locals, globals and the stack.
- The process image is only the Wasm linear memory
- etc.

For experimenting, a custom tool has been built and mimics [gdb]: [wasmgdb].

## Format

The coredump blob starts with the numbers of frame recorded and the combined size of all frames, then the frames themself.

```
coredump ::= framecount:u32 size:u32 cont:frame*
```

> implementer note: since the `frame` struct doesn't have a fixed size, the `size` value can be used to append new frames because it's pointing at the end of the `coredump`.

```
frame ::= funcidx:u32 codeoffset:u32 locals:vec(local) globals:vec(global) stack:vec(stack) reserved:u32
```

The `reserved` bytes are decoded as an empty vector.

`vec` (same encoding as [Wasm Vectors]):
```
vec(B) ::= n:u32 cont:B
```

## Useful links

- [ELF coredump]

[Wasm Vectors]: https://webassembly.github.io/spec/core/binary/conventions.html#binary-vec
[ELF coredump]: https://www.gabriel.urdhr.fr/2015/05/29/core-file/
[Core dump on Wikipedia]: https://en.wikipedia.org/wiki/Core_dump
[gdb]: https://linux.die.net/man/1/gdb
[wasm-edit coredump]: https://github.com/xtuc/wasm-edit/blob/main/src/coredump.rs
[wasm-edit]: https://github.com/xtuc/wasm-edit
[wasmgdb]: https://github.com/xtuc/wasmgdb
[DWARF]: https://yurydelendik.github.io/webassembly-dwarf
