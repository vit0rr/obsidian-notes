What do we know about bytecode?
- It's made up of instructions.
	The instructions themselves are a series of bytes.
	A single instruction consists of an opcode and an optional number of operands.
An opcode is exactly one byte wide, has an arbitrary but unique value and is the first byte on the instruction.

Starting to define Monkey bytecode format:
```go
package code

type Instructions []byte // byte is an alias for uint8.

type Opcode byte
```

Instructions are a slice of bytes and an Opcode is a byte. They match our past description about what we know about bytecodes. 

Now that we have definitions for Opcode and Instructions, I can define the first opcode, the one that tells the VM to push something on the stack. And here's a surprise: the opcode won't have "push" in its name - it won't be solely about pushing things.

To compile the Monkey expression `1 + 2`, I want to generate three different instructions, two of which tell the VM to push `1` and `2` on to the stack. A first instinct might tell me to implement these by defining a "push" instruction with an integer as its operand with the idea being that the VM then takes the integer operand and pushes it on to the stack. For integers it would work, because I could encode them and put them directly into the bytecode, but for string literals, for example, putting those into the bytecode is also possible, since it's just made of bytes, but it would also be a lot of bloat and would sooner or later become unwieldy - that's because string literals can have:
- Variable size: strings can be any length, like "a", or "a more extensive text". 
- I'd have multiple copies of the same string if it appears several times
- Bytecode loading performance would be impacted

That's where the ideia of `constants` come into play. In this context, "constants" is short for "constant expressions" and refers to expressions whose value doesn't change. Is `constant` and can be determined at `compile time`.

![[starting-with-bytes.png]]

That means I don't need to run the program to know what these expressions evaluate to. A compiler can fin them in the code and store the value they evaluate to. After that, It can reference the constants in the instructions it generates, instead of embedding the value directly in them. And while "reference" sounds like a special data type, it's far easier than that. A plain integer does the job just fine and can serve as an index into a data structure that holds all constants, often called a constant pool. For example:
```text
// Instead of having bytecode like:
PUSH 987654321 // Takes up a lot of space if the number is big
PUSH "Hello World" // Takes even more space for string

// I can have:
PUSH_CONST 0 // Where 0 is just an index into the constant pool
PUSH_CONST 1 // Much more compact

Constant Pool:
[0] -> 987654321
[1] -> "Hello World"
```
- More compact bytecode (indices are smaller than full values)
- Deduplication (the same constant only needs to be stored once)
- Better memory usage
- Easier to manage complex constants like strings

And that's exactly what our compiler is going to do. When I come across an integer literal (it means, a constant expression) while compiling, I'll evaluate it and keep track of resulting `*object.Integer` by storing it in memory and assigning it a number. In the bytecode instructions I'll refer to the `*object.Integer` by this number. After we're done compiling and pass the instructions to the VM for execution, we'll also hand over all the constants we've found by putting them in a data structure - the constant pool - where the number that has been assigned to each constant can be used as an index to retrieve it.

Back to the first opcode.
It's called `OpConstant` and it has one operand: the number I previously assigned to the constant. When the VM executes `OpConstant` it retrieves the constant using the operand as an index and pushes it on to the stack.
```go
// [...]

const (
	OpConstant Opcode = iota
) 
```
This addition the groundwork for all future `Opcode` definition. Each definition will have an `Op` prefix and the value it refers to will be determined by `iota`. `Iota` will generate increasing `byte` values, because I just don't care about the actual values the opcodes represent. They only need to be distinct from each other and fit in one byte - `iota` makes sure of that for me.

What's missing from this definition is the part that says `OpConstant` has one operand. There's no technical reason for writing this down, since I could share this piece if knowledge implicitly between compiler and VM. For debugging, though, it's handy being able to lookup how many operands and opcode has and what its human-readable name is. In order to achieve that, I'll add proper definitions and some tooling. to te `code` package.
```go
// code/code.go

type Definition struct {
Name string
operandWidths []int

}

var definitions = map[Opcode]*Definition{
OpConstant: {"Opconstant", []int{2}},
}

  

func Lookup(op byte) (*Definition, error) {
def, ok := definitions[Opcode(op)]
if !ok {
return nil, fmt.Errorf("opcode %d undefined", op)
} 

return def, nil
}
```

The `Definition` for an `Opcode` has two fields: `Name` and `OperandsWidths`. `Name` helps to make an `Opcode` readable and `OperandWidths` contains the number of bytes each operand takes up.

The definition for `OpConstant` says that it's only operand in two bytes wide, which makes it an uint16 and limits its maximum value to `65535`, if include `0` the number of representable values is then `65536`.

With this definition in place I can now create the first bytecode instruction. Without any operands involved that would be as simple as adding an `Opcode` to an `Instructions` slice. But in the case of `OpConstant` we need to correctly encode the two-byte operand. For that, I should create a function that allows me to easily create a single bytecode instruction that's made up of an `Opcode` and an optional number of operands. 
```go
// code/code.go
func Make(op Opcode, operands ...int) []byte {
def, ok := definitions[op]
if !ok {
return []byte{}
}

instructionLen := 1

for _, w := range def.operandWidths {
instructionLen += w
}

instruction := make([]byte, instructionLen)

instruction[0] = byte(op)

offset := 1

for i, o := range operands {
width := def.operandWidths[i]

switch width {

case 2:
binary.BigEndian.PutUint16(instruction[offset:], uint16(o))
}
offset += width
}

return instruction
}
```

```go
// code/code_test.go
func TestMake(t *testing.T) {

tests := []struct {

op Opcode

operands []int

expected []byte

}{

{OpConstant, []int{65534}, []byte{byte(OpConstant), 255, 254}}}

  

for _, tt := range tests {

instructions := Make(tt.op, tt.operands...)

  

if len(instructions) != len(tt.expected) {

t.Errorf("instructions has wrong length. want=%d, got=%d",

len(tt.expected), len(instructions))

}

  

for i, b := range tt.expected {

if instructions[i] != tt.expected[i] {

t.Errorf("wrong byte at pos %d. want=%d, got=%d",

i, b, instructions[i])

}

}

}

}
```

For now, I only pass `OpConstant` and the operand `65534` to Make. Then expect to get back a `[]byte` holding three bytes. Of these three, the first one has to be the opcode, `OpConstant` and the other two should be the big-endian encoding of `65534`. And that's also why use `65534` instead of `65535`:
```text
65534 in decimal = 1111 1111 1111 1110 in binary
                 = 0xFF 0xFE in hexadecimal (two bytes)
```

Big-endian means "most significant byte first". Like reading left-to-right:
- In big-endian: 65534 = [0xFF, 0xFE]
- In little-endian: 65534 = [0xFE, 0xFF]
And `65535` both bytes are the same ([0xFF, 0xFF]) - can't tell the order.

The expected output is 3 bytes:
```text
[OpConstant, 0xFF, 0xFE]
 ^           ^      ^
 |           |      Second byte of 65534
 |           First byte of 65534
 The instruction opcode
```

The first thing I'm doing here is to find out how long the resulting instruction is going to be. That allows me to allocate a byte slice with the proper length. Note that I don't use the `Lookup` function to get to the definition, which gives me a much more usable function signature for `Make` in the tests later on.

As soon as we have the final value of `instructionLen` , we allocate the instruction `[]byte` and add the `Opcode` as its first byte - by casting it into one. Then comes the tricky part: I'm iterate over the defined `OperandsWidths`, take the matching element from `operands`and put it in the instructions. I do that by using a `switch` statement with a different method for each operand, depending on how wide the operand is.

I'll need to extend this `switch` as I define additional `Opcodes`. For now, I only make sure that a two-byte operand is encoded in big endian. After encoding the operand, I increment `offset` by its `width`and the next iteration of the loop. Since the `OpConstant`opcode in the test case has only one operand, the loop performs only one iteration before `Make` returns `instruction`.

And that's it! I created my first bytecode instruction!