It's possible to teach types to print themselves in Go by giving them a `String()` method. And that's also true for bytecode instructions.
```go
// code/code_test.go

func TestInstructionsString(t *testing.T) {
	instructions := []Instructions{
		Make(OpConstant, 1),
		Make(OpConstant, 2),
		Make(OpConstant, 65535),
	}

	expected := `0000 OpConstant 1
0003 OpConstant 2
0006 OpConstant 65535
`

	concatted := Instructions{}
	for _, ins := range instructions {
		concatted = append(concatted, ins...)
	}

	if concatted.String() != expected {
		t.Errorf("instructions wrongly formatted.\nwant=%q\ngot=%q", expected, concatted.String())
	}
}
```

And that's what I expected from `Instructions.String` method. There's a counter at the start of each line, telling which bytes I'm looking at, there are the opcodes in their human-readable form, and then there are the decoded operands.

```go
// code/code.go

func (ins Instructions) String() string {
	return ""
}
```

Return blank string here because that gives the compiler something to chew on and the ability to run tests again - and this test fails, of course.
```text
_want="0000 OpConstant 1\n0003 OpConstant 2\n0006 OpConstant 65535\n"_
_got=""_
```

And it's more useful than an `undefined: String`compiler error that stops from running the tests, because we now I need to write another test and run it.
This other test is for a function that will be the heart of `Instructions.String`. Its name is `ReadOperands`:
```go
// code/code_test.go

func TestReadOperands(t *testing.T) {
	tests := []struct {
		op Opcode
		operands []int
		bytesRead int
	}{
		{OpConstant, []int{65535}, 2}
	}

	for _, tt := range tests {
		instruction := Make(tt.op, tt.operands...)

		def, err := Lookup(byte(tt.op))
		if err != nil {
			t.Fatalf("definition not found: %q\n", err)
		}
	
		operandsRead, n := ReadOperands(def, instruction[1:])
		if n != tt.bytesRead {
			t.Fatalf("n wrong. want=%d, got=%d", tt.bytesRead, n)
		}

		for i, want := range tt.operands {
			if operandsRead[i] != want {
				t.Errorf("operand wrong. want=%d, got=%d", want, operandsRead[i])
			}
		}
	}
}
```

`ReadOperands`is supposed to be `Make's`counterpart. Whereas `Make`encodes the operands of a bytecode instruction it's the job of `ReadOperands`to encode them.

In `TestReadOperands`I `Make`a fully-encoded instruction and pass its definition to `ReadOperands`, along with the subslice of the instruction containing the operands. `ReadOperands`should then return the decoded operands and tell how many bytes it read to do that. So, I'm going to extend the `tests`table as soon as I have more opcodes and different instruction types.

```go
// code/code.go

func ReadOperands(def *Definition, ins Instructions) ([]int, int) {
	operands := make([]int, len(def.OperandWidths))
	offset := 0

	for i, width := range def.OperandWidths {
		switch width {
			case 2:
				operands[i] = int(ReadUint16(ins[offset:]))
		}
	
		offset += width
	}

	return operands, offset
}

func Readuint16(ins Instructions) uint16 {
	return binary.BigEndian.Uint16(ins)
}
```

Just like in `Make`, I use the `*Definition`of an opcode to find out how wide the operands are and allocate a slice with enough space to gold them. We then go through the `Instructions`slice and read in and convert as many bytes as defined in the definition. 
ReadUint16 is a separate and public function, because it can be used directly by the VM, allowing me to skip the definition lookup required by `ReadOperands`.

The tests of course are still failing, because it is still chewing on the blank string.
```go
// code/code.go

func (ins Instructions) String() string {
	var out bytes.Buffer

	i := 0
	for i < len(ins) {
		def, err := Lookup(ins[i])
		if err != nil {
			fmt.Fprintf(&out, "ERROR: %s\n", err)
			continue
		}

		operands, read := ReadOperands(def, ins[i+1:])
	
		fmt.Fprintf(&out, "%04d %s\n", i, ins.fmtInstruction(def, operands))
	
		i += 1 + read
	}

	return out.String()
}
```

```go
func (ins Instructions) fmtInstruction(def *Definition, operands []int) string {
	operandCount := len(def.OperandWidths)
	if len(operands) != operandCount {
		return fmt.Sprintf("ERROR: operand len %d does not match defined %d\n",
		len(operands), operandCount)
	}
	switch operandCount {
		case 1:
			return fmt.Sprintf("%s %d", def.Name, operands[0])
	}

	return fmt.Sprintf("ERROR: unhandled operandCount for %s\n", def.Name)
}
```

And now, the `code`tests pass. The mini-disassembler works. And about the compiler:
```text
_$ go test ./compiler_
_--- FAIL: TestIntegerArithmetic (0.00s)_
_compiler_test.go:31: testInstructions failed: wrong instructions length._
_want="0000 OpConstant 0\n0003 OpConstant 1\n"_
_got =""_
_FAIL_
_FAIL monkey/compiler 0.008s_
```

Not looks better. It's not "x00 x00 x00 x00..." anymore.