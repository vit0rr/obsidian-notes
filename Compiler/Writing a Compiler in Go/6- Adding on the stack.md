To actually add the Monkey expression *1 + 2*, I'll need to implement a new opcode - for now, there's just **OpConstant**.
The new opcode is called **OpAdd** and tells the VM to pop the two topmost elements of the stack, add them together and push the result back on to the stack. In contrast to **OpConstant**, it doesn't have any operands. It's simply one byte, a single opcode:
```go
// code/code.go
var definitions = map[Opcode]*Definition{
	OpConstant: {"OpConstant", []int{2}},
	OpAdd:      {"OpAdd", []int{}},
}

const (
	OpConstant Opcode = iota
	OpAdd
)
```

Note that the **Definition** holds an empty slice to signify that **OpAdd** doesn't have any operands. And now I need to make sure that my tooling can handle an opcode without operands.
```go
func TestMake(t *testing.T) {
	tests := []struct {
		// [...]
	}{
		// [...]
		{OpAdd, []int{}, []byte{byte(OpAdd)}},
	}
    // [...]
```

This new test make sure that **Make** knows how to encode a single **Opcode** into a byte slice. And it does:
```shell
➜ go test ./code   
ok      github.com/vit0rr/mumu/code     0.533s
```

That means that I can now use **Make** to test whether the **Instructions.String** method can also handle **OpAdd**:
```go
// code/code_test.go
func TestInstructionsString(t *testing.T) {
	instructions := []Instructions{
		Make(OpAdd),
		Make(OpConstant, 2),
		Make(OpConstant, 65535),
	}

	expected := `0000 OpAdd
0001 OpConstant 2
0004 OpConstant 65535
```

And... The test fails:
```shell
➜ go test ./code
--- FAIL: TestInstructionsString (0.00s)
    code_test.go:54: instructions wrongly formatted.
        want="0000 OpAdd\n0001 OpConstant 2\n0004 OpConstant 65535\n"
        got="0000 ERROR: unhandled operandCount for OpAdd\n\n0001 OpConstant 1\n0004 OpConstant 2\n0007 OpConstant 65535\n"
FAIL
FAIL    github.com/vit0rr/mumu/code     0.532s
FAIL
```

But the error message points in the right direction. I need to extend the **switch** in the **Instructions.fmtInstruction** method to handle opcodes with no operands:
```go
// code/code.go
func (ins Instructions) fmtInstruction(def *Definition, operands []int) string {
	// [...]
	switch operandCount {
	case 0:
		return def.Name
	case 1:
		return fmt.Sprintf("%s %d", def.Name, operands[0])
	}

	return fmt.Sprintf("ERROR: unhandled operandCount for %s\n", def.Name)
}
```

```shell
mumu on  main [$!] via 🐹 v1.23.2 
➜ go test ./code
ok      github.com/vit0rr/mumu/code     0.545s
```

And since **OpAdd** doesn't have operands, I don't need to change **ReadOperands**. Now, **OpAdd** is fully defined and ready to be used in the compiler.

Now, it's time to fix the **TestIntegerArithmetic**:
```go
// compiler/compiler_test.go
func TestIntegerArithmetic(t *testing.T) {
	tests := []compilerTestCase{
		{
			input:             "1 + 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpAdd),
			},
		},
	}
```
Now the expected is correct. Two **OpConstant** instructions to puh the two constants on the stack and then an **OpAdd** instruction that should cause the VM to add them together. Since I only updated my tools but not yet the compiler, tests tells which instruction we're not emitting:
```shell
--- FAIL: TestIntegerArithmetic (0.00s)
    compiler_test.go:101: testInstructions failed: _wrong instructions length.
        want="0000 OpConstant 0\n0003 OpConstant 1\n0006 OpAdd\n"
        got="0000 OpConstant 0\n0003 OpConstant 1\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.504s
FAIL
``` 

Now I need to emit an **OpAdd** instruction. And since I already came across an `*ast.InfixExpression`in the compiler's `Compile`method, I know where to do that:
```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
    // [...]

	case *ast.InfixExpression:
		err := c.Compile(node.Left)
		if err != nil {
			return err
		}

		err = c.Compile(node.Right)
		if err != nil {
			return err
		}

		switch node.Operator {
		case "+":
			c.emit(code.OpAdd)
		default:
			return fmt.Errorf("unknow operador %s", node.Operator)
		}
```

In the new switch, I check the **Operator** field of **InfixExpression** node. In case I have a **+** at hand I **c.emit** an **OpAdd** instruction. Then, a default case that returns erros, to be safe.

And now tests pass, because the compiler is now able to emit **OpAdd** instructions:
```shell
➜ go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.822s
```

Now, let's implement **OpAdd** to VM too:
```go
// vm/vm_test.go
func TestIntegerArithmetic(t *testing.T) {
	tests := []vmTestCase{
		{"1", 1},
		{"2", 2},
		{"1 + 2", 3},
	}

	runVmTests(t, tests)
}
```
And as expected, I got an error telling that got 2 and want 3. That's because now I have to actually do something with the integers I pushed on to the stack. And the first thing I need to do to add two numbers together is pop the operands off the stack:
```go
// vm/vm.go
func (vm *VM) pop() object.Object {
	o := vm.stack[vm.sp-1]
	vm.sp--
	return o
}
```

First I take the element from the top of the stack, located at `vm.sp-1`, and put it on the side. Then decrement `vm.sp`, allowing the location of element that was just popped off being overwritten eventually.

But, in order to use the **pop** method I need first to add the "decode" part for the new **OpAdd** instruction. But since that's not really worth mentioning on its own, here it is with the first part of the "execute":
```go
// vm/vm.go
func (vm *VM) Run() error {
	for ip := 0; ip < len(vm.instructions); ip++ {
		op := code.Opcode(vm.instructions[ip])

		switch op {
        // [...]

		case code.OpAdd:
			right := vm.pop()
			left := vm.pop()
			leftValue := left.(*object.Integer).Value
			rightValue := right.(*object.Integer).Value
		}
	}

	return nil
}
```

After extending the "decode" part by adding a new case for **code.OpAdd**, I'm ready to implement the operation itself, the "execute". In this case I start by popping the operands off the stack and unwrapping their values into **leftValue** and **rightValue**. 

And note that I'm assume that the **right** operand of the infix operator is the last one to be pushed on to the stack. Here is where subtle bugs can creep in. In the case of **+**, the order of the operands doesn't matter, so the implicitness is not an immediate problem. That was just the start of the implementation of **OpAdd** and the VM test is still failing, so let's finish:
```go
// vm/vm.go
// [...]
		case code.OpAdd:
			right := vm.pop()
			left := vm.pop()
			leftValue := left.(*object.Integer).Value
			rightValue := right.(*object.Integer).Value

			result := leftValue + rightValue
			vm.push(&object.Integer{Value: result})
		}
        // [...]
```

I'm just adding **left** and **right** values together, turn the result into an **object.Integer** and push that on to the stack. And now, the tests pass:
```shell
➜ go test ./vm                
ok      github.com/vit0rr/mumu/vm       0.534s
```

It means that now I'm successfully compiled and executed the Monkey expression **1 + 2**.