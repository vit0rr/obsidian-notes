**I** need to answer a very concrete question: how do I get my VM to execute different bytecode instructions based on a condition?

Just to remember, when I come across an `*ast.IfExpression` in the evaluator package's `Eval` function, I evaluate its `Condition` and check the result with the `isTruthy` function. In case the value is truthy, I `Eval` the `Consequence` of the `*ast.ifExpression`. If it's not truthy, and the `*ast.IfExpression` has an `Alternative`, I `Eval` that instead. but if I don't have an `Alternative`, I return an `*object.Null`.
Implementing conditionals took us around 50 lines of code, because I had the AST nodes in my hands. I could decide which side of the `*ast.IfExpression` to evaluate, because I had both available to me in the evaluator.

And that's not the case anymore. Instead of walking down the AST and executing it at the same time, I now turn the AST into bytecode and flatten it. Flatten because bytecode is a sequence of instructions and there are no child nodes I can choose to walk down or not. That bring me back to the hidden main question how do I represent conditionals in bytecode?

Imagine this code:
```rust
if (5 > 2) {
	30 + 20
} else {
	50 - 25
}
```

Well, I already know how to represent `5 > 2` in bytecode:
![[load-opconstant.png]]

The same for `30 + 20`, but of course the OpGreaterThan will be OpAdd, and the Constants will Load the 30 and 20.

But how do I tell the machine to either execute the one part or the other part, depending on the result of the OpGreatherThan instruction?
![[branch.png]]

Of course if I take these instructions and pass them to the VM as a flat sequence, the VM would execute all of them, one after the other, happily incrementing its instruction pointer, fetching, decoding and executing without a care in the world, no decision or branches in sight - and that's exactly what I don't want.

### Jumps
Jumps are instructions that tell machines to jump to other instructions. They're used to implement branching (conditionals) in machine code, giving them the name "branch instructions". And with "machine code" I mean the code that computers execute but also the bytecode virtual machine run on. i.e. jumps are instructions that tell the VM to change its instruction pointer to a certain value. 

![[flat-memory.png]]

The operand of `JUMP_IF_NOT_TRUE` is now 0008. That's the index of the `OpConstant 4` instruction to which the VM should jump in case the condition is not true. The operand of `JUMP_NO_MATTER_WHAT` is 0011, which is the index of the instruction following the whole conditional.
I'll define two jump opcodes: one comes with a condition("jump only if not true") and one does not("just jump"). They'll both have one operand, the index of the instruction where the VM should jump to.

### Compiling conditionals
Imagine I'm in my compiler's recursive `Compile` method, having just called `Compile` again, passing in the `.Condition` field of an `*.ast.ifExpression`. The condition has been successfully compiled and we've emitted the translated instructions. Now I want to emit the jump instruction that tells the VM to skip to the consequence of the conditional if the value on the stack is not true. Which operand do I hive this jump instruction? Where do I tell the VM to jump to? I don't know yet. Since I haven't compiled the consequence or the alternative branch yet, I don't know how many instructions I'm going to emit, which means I don't know how many instructions I have to  jump over. That's the challenge.

```go
// code/code.go

const (
	OpConstant Opcode = iota
	OpPop
	OpAdd
	OpSub
	OpMul
	OpDiv
	OpTrue
	OpFalse
	OpEqual
	OpNotEqual
	OpGreaterThan
	OpOr
	OpMinus
	OpBang
	OpJumpNotTruthy
	OpJump
)

var definitions = map[Opcode]*Definition{
	OpConstant:      {"OpConstant", []int{2}},
	OpPop:           {"OpPop", []int{}},
	OpAdd:           {"OpAdd", []int{}},
	OpSub:           {"OpSub", []int{}},
	OpMul:           {"OpMul", []int{}},
	OpDiv:           {"OpDiv", []int{}},
	OpTrue:          {"OpTrue", []int{}},
	OpFalse:         {"OpFalse", []int{}},
	OpEqual:         {"OpEqual", []int{}},
	OpNotEqual:      {"OpNotEqual", []int{}},
	OpGreaterThan:   {"OpGreaterThan", []int{}},
	OpOr:            {"OpOr", []int{}},
	OpMinus:         {"OpMinus", []int{}},
	OpBang:          {"OpBang", []int{}},
	OpJumpNotTruthy: {"OpJumpNotTruthy", []int{2}},
	OpJump:          {"OpJump", []int{2}},
}
```

The operand of both opcodes is 16-bit wide. That's the same width as the operand of `OpConstant` has, which means I don't have to extend my tooling in the `code` package to support it. I'm now ready to write a first test. I'll try first only handle a conditional without an `else` part:

```go
// compiler/compiler_test.go

func TestConditionals(t *testing.T) {
	tests := []compilerTestCase{
		{
			input: `
			if (true) { 10 }; 3333;
			`,
			expectedConstants: []interface{}{10, 3333},
			expectedInstructions: []code.Instructions{
				// 0000
				code.Make(code.OpTrue),
				// 0001
				code.Make(code.OpJumpNotTruthy, 7),
				// 0004
				code.Make(code.OpConstant, 0),
				// 0007
				code.Make(code.OpPop),
				// 0008
				code.Make(code.OpConstant, 1),
				// 0011
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

Once parsed, the input turns into an `*ast.IfExpression` with a Condition and a Consequence. The condition is the boolean literal true and the Consequence if the integer literal 10. Both are intentionally simple Monkey expressions, because in this test case I do not care about the expressions themselves. What I care about are the jump instructions the compiler emits and that hey have correct operands.

The first instruction I expect the compiler to emit is an OpTrue (push vm.True on to the stack). That's the Condition. Then it should emit an OpJumpNotTruthy instruction that causes the VM to jump over the Consequence, with the Consequence being the OpConstant instruction that loads the integer 10 to the stack.

The OpPop instruction (offset 0007) it's not part of the Consequence. It's there because conditionals in Monkey are expressions - if (true) {10} evaluates to 10 - and stand-alone expressions whose value is unused are wrapped in an `*ast.ExpressionStatement`. And those we compile with an appended OpPop instruction in order to clear the VM's stack. The first OpPop is thus the first instructioin after the whole conditional, which makes its offset the location where OpJumpNotTruthy needs to jump to in order to skip the consequence.

The 3333 are just a point of reference. It's not strictly needed, but in order to make sure that my jump ofsets are correct, it helps to have one expression in the code which I can easily find among the resulting instructions and used as signpost that tells us where we shouldn't jump to. Of course, the OpConstant 1 instruction that loads 3333 is also followed by an OpPop instruction, since it's an expression statement.

Tests of course not pass:
```shell
go test ./compiler 
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:280: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
        got="0000 OpPop\n0001 OpConstant 0\n0004 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.493s
FAIL
```

Neither the condition nor the consequence of the conditional are compiled. In fact, the whole `*ast.IfExpression` is skipped by the compiler. I can fix the first issue, the condition not being compiled, by extending the compiler's Compile method:

```go
// compiler/compiler.go

func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	// [...]

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

	// [...]

	return nil
}
```

```shell
go test ./compiler
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:280: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
        got="0000 OpTrue\n0001 OpPop\n0002 OpConstant 0\n0005 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.451s
FAIL
```

The OpTrue instruction is there, as are the last three, the OpPop following the `*ast.IfExpression`, the `OpConstant` to load the 3333 and the `OpCode` following that, all in the correct order. All that's left to do now is emit the `OpJumpNotTruthy` instruction and the instructions to represent the node.Consequence.

The challenge now is to emit an `OpJumpNotTruthy` instruction with an offset pointing right after the instructions of the node.Consequence - before compiling the node.Consequence.
For now, I'll put garbage to the jump and fix it later.

```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}
```

Yes, 9999.
But I really want to emit an OpJumpNotTruthy instruction with a garbage offset and then compile the node.Consequence. For now, it should get a lot more correct instruction in my test.

But, no, I only get one more right and that's the OpJumpNotTruthy instruction itself:
```shell
go test ./compiler 
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:280: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
        got="0000 OpTrue\n0001 OpJumpNotTruthy 9999\n0004 OpPop\n0005 OpConstant 0\n0008 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.568s
FAIL
```

While I have the OpJumpNotTruthy 9999 instruction, I'm apparently not yet compiling the Consequence. That's because it's an `*ast.BlockStatement`, which my compiler doesn't know about yet. in order to get it compiled, I need to extend the Compile method by another case branch:

```go
// compiler/compiler.go

	case *ast.BlockStatement:
		for _, s := range node.Statements {
			err := c.Compile(s)
			if err != nil {
				return err
			}
		}
		```

That's exactly the same snipped of code I already have in the case b ranch for `*ast.Program`. And it works.
```shell 
go test ./compiler
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:280: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
        got="0000 OpTrue\n0001 OpJumpNotTruthy 9999\n0004 OpConstant 0\n0007 OpPop\n0008 OpPop\n0009 OpConstant 1\n0012 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.518s
FAIL
```

Getting closer.
But, besides the bogus 9999 offset, which I didn't expect to magically disappear, there's a new issue visible in the output, a far more subtle one: there is an additional OpPop instruction generated by the compiler, at position 0007. its origin is the compilation of node.Consequence - an expression statement.
I need to get ride of this OpPop, because I do want the consequence and the alternative of a conditional to leave a value on the stack. Otherwise, I couldn't do this:
```rust
let result = if (5 > 3) { 5 } else { 3 };
```

That's valid Monkey code and it won't work if I emit an OpCode after the last expression statement in the node.Consequence. The value produced by the consequence would be popped off the stack, the expression wouldn't evaluate to anything, and the let statement would end up without a value on the right side of its =.

What makes fixing this tricky is that I only want to get rid of the last OpCode instruction in the node.Consequence:
```rust
if (true) {
	3;
	2;
	1;
}
``` 

What I want here is the 3 and the 2 to be popped of the stack, but the 1 should be kept around so the whole conditional evaluates to 1.
So, first, I need to change the compiler to keep track of the last two instructions I emitted, including their opcode and the position they were emitted to. For that, I need  a new type and two more fields on the compiler.

```go
// compiler/compiler.go
type Compiler struct {
	instructions code.Instructions
	constants    []object.Object

	lastInstruction     EmittedInstruction
	previousInstruction EmittedInstruction
}

func New() *Compiler {
	return &Compiler{
		instructions: code.Instructions{},
		constants:    []object.Object{},

		lastInstruction:     EmittedInstruction{},
		previousInstruction: EmittedInstruction{},
	}
}

// [...]

type EmittedInstruction struct {
	Opcode   code.Opcode
	Position int
}
```

lastInstruction is the very last instruction I emitted and previousInstruction is the one before that. I'll see why I need to keep track of both in a moment. For now, I change the compiler's emit method to populate both fields:
```go
// compiler/compiler.go

func (c *Compiler) setLastInstruction(op code.Opcode, pos int) {
	previous := c.lastInstruction
	last := EmittedInstruction{Opcode: op, Position: pos}

	c.previousInstruction = previous
	c.lastInstruction = last
}

func (c *Compiler) addConstant(obj object.Object) int {
	c.constants = append(c.constants, obj)
	return len(c.constants) - 1
}

func (c *Compiler) emit(op code.Opcode, operands ...int) int {
	ins := code.Make(op, operands...)
	pos := c.addInstruction(ins)

	c.setLastInstruction(op, pos)

	return pos
}
```

With this in place, I can check opcode of the last emitted instruction in a type-safe way, without having to cast from and to bytes. And that's exactly what I'm going to do. After compiling the node.Consequence of the `*ast.IfExpression` I check whether the last instruction I emitted was an OpPop instruction and if so, I remove it:
```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}

		if c.lastInstructionIsPop() {
			c.removeLastPop()
		}
```

And it uses two new helper methods:
```go
// compiler/compiler.go

func (c *Compiler) lastInstructionIsPop() bool {
	return c.lastInstruction.Opcode == code.OpPop
}

func (c *Compiler) removeLastPop() {
	c.instructions = c.instructions[:c.lastInstruction.Position]
	c.lastInstruction = c.previousInstruction
}
```

lastInstructionIsPop checks whether the opcode of the last instruction is OpPop and removeLastPop shortens c.instruction to cut off the last instruction. After that, it sets c.lastInstruction to c.previousInstruction. And that's why I need to keep track of both of them, so c.lastInstruction doesn't go out of sync once I cut off the last OpPop instruction.
```shell
go test ./compiler
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:280: testInstructions failed: _wrong instruction at 2.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
        got="0000 OpTrue\n0001 OpJumpNotTruthy 9999\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.508s
FAIL
```

Now I have the correct number of instructions and the right opcodes, but the 9999 are still wrong. The way I took care of the superfluous OpPop instruction points me into the right direction by making one thing clear: the instruction I emit are not set in stone. I can change them.

Instead of removing my c.emit(code.OpJumpNotTruthy, 9999) call, I'll leave it exactly as it is. I won't even change the 9999. Instead, I'll use Position field of the c.lastInstruction again. That allows me to go back to the OpJumpNotTruthy instruction I emitted and change the 9999 into the real operand. And when do I do that? That's the kicker, the beautiful bit. I'll modify the operand after I compiled the node.Consequence. At that point, I'll know how far the VM has to jump and have the correct offset with which to replace the 9999.
This is caled back-patching and common is compiler's such as mine, that only traverse the AST once and are thus called single-pass compilers. More advanced compilers might leave the target of the jump instructions empty until they know how far to jump and then do a second pass over the AST (or another IR) and fill in the targets.
In other words, I'll keep on emitting the 9999, while remembering where I put it. Once I know where I need to jump to, I'll go back to the 9999 and change it to the correct offset. 

The first thing I need is a tiny method to replace an instruction at an arbitrary offset in the instructions slice:
```go
// compiler/compiler.go

func (c *Compiler) replaceInstruction(pos int, newInstruction []byte) {
	for i := 0; i < len(newInstruction); i++ {
		c.instructions[pos+i] = newInstruction[i]
	}
}
```

I'll use replaceInstruction in another method that allows me to replace the operand of an instruction:
```go
// compiler/compiler.go

func (c *Compiler) changeOperand(opPos int, operand int) {
	op := code.Opcode(c.instructions[opPos])
	newInstruction := code.Make(op, operand)

	c.replaceInstruction(opPos, newInstruction)
}
```

The changeOperand method recreates the instructions with the new operand and uses replaceInstruction to swap the old instruction for the new one - including the operand.

The underlying assumption here is that I only replace instructions of the same type, with the same non-variable length. If that assumption no longer holds, I'd have to tread far ore carefully here and update c.lastInstruction and c.previousInstruction accordingly. I can see how another IR that's type-safe and independent of the byte-size of encoded instruction some in handy once the compiler and the instructions it emits grow more complex.

My solution, though, still fits my need and all in all is not a lot of code. Now, let's change the compiler switch branch:
```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		jumpNotTruthyPos := c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}

		if c.lastInstructionIsPop() {
			c.removeLastPop()
		}

		afterConsequencePos := len(c.instructions)
		c.changeOperand(jumpNotTruthyPos, afterConsequencePos)
```

The first change is saving the return value of c.emit to jumpNotTruthyPos. That's the position at which I can find the OpJumpnotTruthy instruction later on. Later on means right after the check for and possible removal of a OpPop instruction. After that, len(c.instructions) gives me the offset of the next-to-be-emitted instruction, which is where I want to jump to in case I don't execute the Consequence of the conditional because the value on top of the stack is not truthy. That's why I save it to afterConsequencePos, to give it a telling name.
After that, I use the new changeOperand method to get rid of the 9999 operand of the OpJumpNotTruthy instruction, which is located at jumpNotTruthyPos, and replace it with the correct afterConsequencePos.
And now, the test pass:
```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.694s
```

But it only knows how to compile the consequence. It doesn't know how to compile a conditional with both a consequence and an alternative else-branch:
```go
// compiler/compiler_test.go

		{
			input: `
			if (true) { 10 } else { 20 }; 3333;
			`,
			expectedConstants: []interface{}{10, 20, 3333},
			expectedInstructions: []code.Instructions{
				// 0000
				code.Make(code.OpTrue),
				// 0001
				code.Make(code.OpJumpNotTruthy, 10),
				// 0004
				code.Make(code.OpConstant, 0),
				// 0007
				code.Make(code.OpJump, 13),
				// 0010
				code.Make(code.OpConstant, 1),
				// 0013
				code.Make(code.OpPop),
				// 0014
				code.Make(code.OpConstant, 2),
				// 0017
				code.Make(code.OpPop),
			},
		},
```

This is similar to the previous test case in TestConditionals, except that the input now contains not only the consequence of the conditional, but also the alternative: else { 20 }.
The first part of the test is equal to the last one. But the second, differs. 

Ad the next opcode, I expect an OpJump, the opcode for an unconditional jump instruction. It has to be there because if condition is truthy the VM should only execute the consequence and not the alternative. To stop that from happening the OpJump instruction tells the VM to jump over the alternative.
The OpJump should then be followed by instructions that makes up the alternative. In my test case, that's the OpConstant instruction that loads 20 on to the stack.
What the test tells me:
```shell
go test ./compiler
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:304: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 10\n0004 OpConstant 0\n0007 OpJump 13\n0010 OpConstant 1\n0013 OpPop\n0014 OpConstant 2\n0017 OpPop\n"
        got="0000 OpTrue\n0001 OpJumpNotTruthy 7\n0004 OpConstant 0\n0007 OpPop\n0008 OpConstant 1\n0011 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.507s
FAIL
```

What I have here is the condition, then the OpCode following the whole conditional and the pushing and popping of the 3333. What's missing is the OpJump at the end of the consequence and the instructions representing the alternative. The good news is that I already have all the tools at hand. I just need to move things around a tiny bit and compile the alternative.
The first step is to wrap my patching of the OpJumpNotTruthy instruction in a condition itself:

```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		jumpNotTruthyPos := c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}

		if c.lastInstructionIsPop() {
			c.removeLastPop()
		}

		if node.Alternative == nil {
			afterConsequencePos := len(c.instructions)
			c.changeOperand(jumpNotTruthyPos, afterConsequencePos)
		} else {
			c.emit(code.OpJump, 9999)

			afterConsequencePos := len(c.instructions)
			c.changeOperand(jumpNotTruthyPos, afterConsequencePos)
		}
```

Preceding this `node.Alternative == nil`  check is the compilation of `node.Consequence` and what this added block translates to is this: only if I have no `node.Alternative` can I jump to here, the current position in `c.instructions`.
But if I do have an `node.Alternative` I need to emit an `OpJump` that becomes part of the consequence and over which the `OpJumpNotTruthy` also has to jump.

The `OpJump` instruction also has a placeholder operand. That means I have to patch it later, but right now it allows us to change the operand of the `OpJumpNotTruthy` instruction to the desired value: the position of the instruction right after the consequence and the `OpJump` instruction.
And why that is the correct operand should be clear by now: the `OpJump` should skip over the "else" branch of the conditional in case the condition was truthy. It's part of the consequence, so to say. And if the condition is not truthy and I need to execute the else branch, I need to use `OpJumpNotTruthy` to jump *after* the consequence, which is after the `OpJump`.
The tests tell me that I'm on the right track:
```shell
go test ./compiler
--- FAIL: TestConditionals (0.00s)
    compiler_test.go:304: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpJumpNotTruthy 10\n0004 OpConstant 0\n0007 OpJump 13\n0010 OpConstant 1\n0013 OpPop\n0014 OpConstant 2\n0017 OpPop\n"
        got="0000 OpTrue\n0001 OpJumpNotTruthy 10\n0004 OpConstant 0\n0007 OpJump 9999\n0010 OpPop\n0011 OpConstant 1\n0014 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.546s
FAIL
```

The operand of `OpJumpNotTruthy` is correct, `OpJump` is in the correct place, only its operand is wrong and the whole alternative is missing. I now have to repeat what I previously did for the consequence:

```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		jumpNotTruthyPos := c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}

		if c.lastInstructionIsPop() {
			c.removeLastPop()
		}

		if node.Alternative == nil {
			afterConsequencePos := len(c.instructions)
			c.changeOperand(jumpNotTruthyPos, afterConsequencePos)
		} else {
			jumpPos := c.emit(code.OpJump, 9999)

			afterConsequencePos := len(c.instructions)
			c.changeOperand(jumpNotTruthyPos, afterConsequencePos)

			err = c.Compile(node.Alternative)
			if err != nil {
				return err
			}

			if c.lastInstructionIsPop() {
				c.removeLastPop()
			}

			afterAlternativePos := len(c.instructions)
			c.changeOperand(jumpPos, afterAlternativePos)
		}
```

```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.607s
```

DONE
I'm compiling conditionals to jump instructions.

### Executing jumps

```go
// vm/vm_test.go

func TestConditionals(t *testing.T) {
	tests := []vmTestCase{
		{"if (true) { 10 }", 10},
		{"if (true) { 10 } else { 20 }", 10},
		{"if (false) { 10 } else { 20 }", 20},
		{"if (1) { 10 }", 10},
		{"if (1 < 2) { 10 }", 10},
		{"if (1 < 2) { 10 } else { 20 }", 10},
		{"if (1 > 2) { 10 } else { 20 }", 20},
	}

	runVmTests(t, tests)
}
```

Half of thee test cases would've been enough. Doesn't hurt me to be abundantly clear about what I want.

I test whether boolean expressions are correctly evaluated by the VM according to Monkey's "truthy" standards and that the correct branch of the conditional is taken. Since conditionals are expressions that produces values, they allow us to infer which branch was executed by testing for the produced value of the whole conditional.

And as neat as the tests are, the error message they produces is nasty:

```shell
➜ go test ./vm      
--- FAIL: TestConditionals (0.00s)
panic: runtime error: index out of range [1792] with length 1 [recovered]
        panic: runtime error: index out of range [1792] with length 1

goroutine 33 [running]:
testing.tRunner.func1.2({0x1002a4b40, 0x14000136000})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1632 +0x1bc
testing.tRunner.func1()
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1635 +0x334
panic({0x1002a4b40?, 0x14000136000?})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/runtime/panic.go:785 +0x124
github.com/vit0rr/mumu/vm.(*VM).Run(0x1400012dd68)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm.go:51 +0x3d8
github.com/vit0rr/mumu/vm.runVmTests(0x1400011b6c0, {0x14000135e78, 0x7, 0x7?})
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:66 +0x294
github.com/vit0rr/mumu/vm.TestConditionals(0x1400011b6c0?)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:166 +0x144
testing.tRunner(0x1400011b6c0, 0x1002b2748)
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1690 +0xe4
created by testing.(*T).Run in goroutine 1
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1743 +0x314
FAIL    github.com/vit0rr/mumu/vm       0.595s
FAIL
```

The VM is tripping over the bytecode because it contains opcode it doesn't know how to decode. That in itself shouldn't be a problem, because unknow opcodes are skipped, but not necessarily their operands. Operands are just integers, and might have the same value as an encoded opcode, which might lead the VM to treat them as such. That's wrong. of course. It's time I introduce my VM to my jump instruction.

I'll start with `OpJump`, because it's the most straightforward jump instruction I have. It has one 16 bit operand that's the offset of the instruction the VM should jump to. That's all I need to know to implement it:
```go
// vm/vm.go

		case code.OpJump:
			pos := int(code.ReadUint16(vm.instructions[ip+1:]))
			ip = pos - 1
		}
```

I use `code.ReadUint16`to decode the operand located right after the opcode. That's step 1. Step 2 is to set the instruction pointer, `ip`, to the target of our jump. Here's where I come across one interesting implementation detail: since I'm on a loop that increments ip with each iteration I need to set ip to the offset right before the one I want. That lets the loop do its work and ip gets set to the value I want in the next cycle. Now, let's to implement OpJumpNotTruthy:

```go
// vm/vm.go
		case code.OpJumpNotTruthy:
			pos := int(code.ReadUint16(vm.instructions[ip+1:]))
			ip += 2

			condition := vm.pop()
			if !isTruthy(condition) {
				ip = pos - 1
			}
		}

	}

	return nil
}

func isTruthy(obj object.Object) bool {
	switch obj := obj.(type) {
	case *object.Boolean:
		return obj.Value

	default:
		return true
	}
}
```

Again ReadUint16 to read in and decode the operand. After that I manually increase ip by two so I correctly skip over the two bytes of the operand in the next cycle. That's not a new - I've already done that when executing OpConstant instructions.

What's new is the rest.
I pop off the topmost stack element and check if it's truthy with the helper function. If it's not, I jump, which means that I set ip to the index of the instruction right before the target, letting the for-loop do its work.

if the value is truthy I do nothing and start another iteration of the main loop. The result is that I'm executing the consequence of the conditional, which is made ip of the instructions right after the OpJumpNotTruthy instruction.

And now, tests passes!
```shell
go test ./vm
ok      github.com/vit0rr/mumu/vm       0.524s
```

My bytecode compiler and VM are now able to compile and execute Monkey conditionals.

### Welcome back, null
Good, but not all good.
```rust
if (false) {10;}
```

What's happens when the condition of a conditional is not true but the conditional itself has no alternative? The answer of my last interpreter, was `*object.Null`, Monkey's null value. That makes sense, because conditionals are expressions and expressions, by definition, produce values. So what does an expression that produce nothing evaluate to? Null. 
Languages needs to represent nothing by something, and in my code, it will be null. Time to introduce `*object.Null` to my compiler and VM and make this type of conditional work properly.

The first thing is to define an `*object.Null` in my VM. Since its value is contant, I can define it as a global variable, just like my previous global definitions of `vm.True` and `vm.False`.

```go
// vm/vm.go

var Null = &object.Null{}
```

I can simply check if an `object.Object` is `*object.Null` by checking whether it's equal to `vm.Null`. I do not have to unwrap it and take a look at its value. 
Now, I'll first write VM tests because the VM tests allow me to express what I want so succinctly:

```go
// vm/vm_test.go

func testExpectedObject(
	t *testing.T,
	expected interface{},
	actual object.Object,
) {
	t.Helper()

	switch expected := expected.(type) {
	// [...]
	case *object.Null:
		if actual != Null {
			t.Errorf("object is not Null. got=%T (%+v)", actual, actual)
		}
	}
}

func TestConditionals(t *testing.T) {
	tests := []vmTestCase{
		// [...]
		{"if (1 > 2) { 10 }", Null},
		{"if (false) { 10 }", Null},
	}

	runVmTests(t, tests)
}
```

But, the test do not pass:
```shell
➜ go test ./vm                              
--- FAIL: TestConditionals (0.00s)
panic: runtime error: index out of range [-1] [recovered]
        panic: runtime error: index out of range [-1]

goroutine 36 [running]:
testing.tRunner.func1.2({0x1050fcc40, 0x1400011e018})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1632 +0x1bc
testing.tRunner.func1()
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1635 +0x334
panic({0x1050fcc40?, 0x1400011e018?})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/runtime/panic.go:785 +0x124
github.com/vit0rr/mumu/vm.(*VM).pop(...)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm.go:252
github.com/vit0rr/mumu/vm.(*VM).Run(0x1400018fd28)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm.go:94 +0x4dc
github.com/vit0rr/mumu/vm.runVmTests(0x14000185860, {0x14000197e38, 0x9, 0x9?})
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:66 +0x294
github.com/vit0rr/mumu/vm.TestConditionals(0x14000185860?)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:173 +0xf0
testing.tRunner(0x14000185860, 0x10510a848)
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1690 +0xe4
created by testing.(*T).Run in goroutine 1
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1743 +0x314
FAIL    github.com/vit0rr/mumu/vm       2.159s
FAIL
```

The cause of this panic are the `OpPop` instructions I emit after the conditionals. Since they produced no value, the VM crashes trying to pop something off the stack. Time to put `vm.Null` on to the stack.

First, I need to define an opcode that tells the VM to put `vm.Null` on the stack. Then, I'll modify the compiler to insert an alternative when a conditional doesn't have one. And the only thing this alternative branch will contain is the new opcode that pushes `vm.Null` on to the stack.

```go
const (
	[...]
	OpNull
)

var definitions = map[Opcode]*Definition{
	// [...]
	OpNull: {"OpNull", []int{}},
}
```


And of course, Null doesn't have any operands.
Now, I need to update a test case in `TestConditionals` and expect to find `OpNull` in the generated instructions. Note that I need to change the first test case, the one in which the conditional doesn't have an alternative.

```go
// compiler/compiler_test.go

func TestConditionals(t *testing.T) {
	tests := []compilerTestCase{
		{
			input: `
			if (true) { 10 }; 3333;
			`,
			expectedConstants: []interface{}{10, 3333},
			expectedInstructions: []code.Instructions{
				// 0000
				code.Make(code.OpTrue),
				// 0001
				code.Make(code.OpJumpNotTruthy, 10),
				// 0004
				code.Make(code.OpConstant, 0),
				// 0007
				code.Make(code.OpJump, 11),
				// 0010
				code.Make(code.OpNull),
				// 0011
				code.Make(code.OpPop),
				// 0012
				code.Make(code.OpConstant, 1),
				// 0015
				code.Make(code.OpPop),
			},
		},
	// [...]
```

New are two instructions in the middle: OpJump and OpNull. OpJump is there to jump over the alternative and now OpNull is the alternative. And since the addition of these two instructions change the index of existing instructions, the operand of OpJumpNotTruthy also has to be changed from 7 to 10.  Of course compiler didn't learn how to insert artificial alternatives on its own.

Now, to fix it, I can make the code in my compiler simpler and easier to understand. I no longer have to check whether to emit OpJump or not, because I always want to do that now. Only sometimes do I want jump over a "real" alternative and sometimes over an OpNull instruction. So, let's change the `case *ast.IfExpression` branch of the `Compile` method.

```go
// compiler/compiler.go

	case *ast.IfExpression:
		err := c.Compile(node.Condition)
		if err != nil {
			return err
		}

		jumpNotTruthyPos := c.emit(code.OpJumpNotTruthy, 9999)

		err = c.Compile(node.Consequence)
		if err != nil {
			return err
		}

		if c.lastInstructionIsPop() {
			c.removeLastPop()
		}

		jumpPos := c.emit(code.OpJump, 9999)

		afterConsequencePos := len(c.instructions)
		c.changeOperand(jumpNotTruthyPos, afterConsequencePos)

		if node.Alternative == nil {
			c.emit(code.OpNull)
		} else {
			err := c.Compile(node.Alternative)
			if err != nil {
				return err
			}

			if c.lastInstructionIsPop() {
				c.removeLastPop()
			}
		}

		afterAlternativePos := len(c.instructions)
		c.changeOperand(jumpPos, afterAlternativePos)
```

Only the second half has been changed: the duplicated patching of the `OpJumpNotTruthy` instruction is gone and in its place I can find the new, readable compilation of a possible `node.Alternative`.

I start by emitting an `OpJump` instruction and updating the operand of the `OpJumpNotTruthy` instruction. That happens whether I have a `node.Alternative` our not. But then I check whether `node.Alternative` is nil and if it is, I emit the new `OpNull` opcode. If it's not nil, I proceed as before: compile `node.Alternative` and then try to get rid of a possible `OpPop` instruction.

After that, I change the operand of the `OpJump` instruction to jump over the freshly-compiled alternative - no matter whether that's just an `OpNull` or more.
```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.474s
```

Now, I need to implement the new OpCode:
```go
// vm/vm.go

		case code.OpNull:
			err := vm.push(Null)
			if err != nil {
				return err
			}
		}
```


That means that I made conditionals with a non-truthy condition put Null on to the stack - the complete behavior of conditionals as described in the interpreter are done. But of course it's not all. Since a conditional is an expression and expressions can be used interchangeably, it follows that any expression can now produce Null in my VM.
For me, the practical implication is that I now have to handle Null in every place where I handle value produced by an expression. Thankfully, most of these places ion my VM - like `vm.ExecuteBinaryOperation` - throw an error if they come across a value they did not expect. But there are functions and methods that now must handle Null explicitly.

The first of these methods are `vm.executeBangOperator`. I can add a test to make sure that it handles Null without blowing up:

```go
// vm/vm_test.go

func TestBooleanExpressions(t *testing.T) {
	// [...]
	{"!(if (false) {5;})", true}}
	
	runVmTests(t, tests)
}
```

With this, I test if a non-true condition and no alternative results in Null, and then, apply bang operator on this Null, turning it into True. Under the hood, this involves `vm.executeBangOperator` and in order to get the test to pass, I need to change it:
```go
// vm/vm.go

func (vm *VM) executeBangOperator() error {
	operand := vm.pop()

	switch operand {
	case True:
		return vm.push(False)
	case False:
		return vm.push(True)
	case Null:
		return vm.push(True)
	default:
		return vm.push(False)
	}
}
```

The negation of Null is now True - as in the interpreter. And the test now pass:
```shell
go test ./vm
ok      github.com/vit0rr/mumu/vm       0.502s
```

Now, here comes the weird part. Since a conditional is an expression and its condition is one too, it follows that I can use a conditional as the condition of another conditional. 

```go
// vm/vm_test.go

func TestConditionals(t *testing.T) {
	tests := []vmTestCase{
		// [...]
		{"if ((if (false) { 10 })) { 10 } else { 20 }", 20},
	}

	runVmTests(t, tests)
}
```

It looks like might be a mess to fix, but I just need to change one place. I need to tell the VM that an `*object.Null` is not isTruthy.
```go
// vm/vm.go

func isTruthy(obj object.Object) bool {
	switch obj := obj.(type) {
	case *object.Boolean:
		return obj.Value

	case *object.Null:
		return false

	default:
		return true
	}
}
```

```shell
go test ./vm
ok      github.com/vit0rr/mumu/vm       0.470s
```

And now done means done.
The implementation of conditionals is now feature complete and I have a Null-safe VM