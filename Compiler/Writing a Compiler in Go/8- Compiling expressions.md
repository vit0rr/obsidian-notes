### Cleaning up the stack
At this point the only thing my compiler and my VM are capable of is adding two numbers. Given 1 + 2, the VM will correctly put 3 on its stack. That's good, but the problem is that the 3 stayus on the stack for all eternity. 

As a quick refresh, I have three types of statements in Monkey: let statements, return statements and expression statements. The problem is not with the expression 1 + 2 itself, but rather where is occurs. It's part of an expression statement. The first two explicitly reuse the values their child-expression nodes produce, expression statements merely wrap expressions so they can occur on their own. The value they produces is not reused, by definition. But now the problem is that I do reuse it, because I involuntarily keep it on the stack. 
```Monkey
1;
2;
3;
```

That's three separate expression statements. And what ends up on the stack? Not just the value produced last, 3, but everything: 1, 2 and 3. If I have a Monkey program consisting of lots of expression statements I could fill up the stack by accident.
To fix this, I need to do two things: First, define a new opcode that tells the VM to pop the topmost element of the stack. Second, emit this opcode after every expression statement.

Let's define the opcode, **OpPop**
```go
// code/code.go
const (
	OpConstant Opcode = iota
	OpAdd
	OpPop
)

var definitions = map[Opcode]*Definition{
	OpConstant: {"OpConstant", []int{2}},
	OpAdd:      {"OpAdd", []int{}},
	OpPop:      {"OpPop", []int{}},
}
```

Just like **OpAdd**, **OpPop** does not have operands. Its only job is to tell the VM to pop the topmost element off the stack and for that it doesn't need an operand.
Now I need to use this opcode to clean the stack after every expression statement. To this, let's start by tests, and change **TestIntegerArithmetic**:
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
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

The only change here is the new line containing the **code.Make(code.OpPop)** call. I assert that the compiled expression statement should be followed by an **OpPop** instruction. The desired behavior can be made even clearer by adding another test with multiple expression statements:
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
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1; 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpPop),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

Note that the **;** that separates the **1** from **2**. Both integers literals are separate expression statements and after each statement an **OpPop** instruction should be emitted. And, ofc, that's not what currently happens. Instead I tell my VM to fill up its stack by loading constants on to it:
```shell
go test ./compiler 
--- FAIL: TestIntegerArithmetic (0.00s)
    compiler_test.go:112: testInstructions failed: _wrong instructions length.
        want="0000 OpConstant 0\n0003 OpConstant 1\n0006 OpAdd\n0007 OpPop\n"
        got="0000 OpConstant 0\n0003 OpConstant 1\n0006 OpAdd\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.526s
FAIL
```

But - that's the good part, in order to fix this and properly clean the stack, all I need to do is to add a single call to **c.emit** to my compiler:
```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
    // [...]
	case *ast.ExpressionStatement:
		err := c.Compile(node.Expression)
		if err != nil {
			return err
		}
		c.emit(code.OpPop)
```

After compiling the **node.Expression** of an `*ast.ExpressionStatement` I emit an **OpPop**. That's all it takes:
```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.499s
```

That's not all it takes. I still need to work left to do, because now I need to tell the VM how to handle this **OpPop** instruction, which would also be a tiny addition of it weren't for my tests.

In VM tests I used **vm.StackTop** to make sure that VM put the correct things on to its stack, but with **OpPop** in play I can't do that anymore. Now, what I wanna assert is that "this should have been on the stack, right before you popped it off, dear VM". In order to do that, I add a test-only method to my VM and get rid of **StackTop**:
```go
// vm/vm.go
func (vm *VM) LastPoppedStackElem() object.Object {
	return vm.stack[vm.sp]
}
```

**vm.sp** always point to the next free slot in **vm.stack**. This is where a new element would be pushed. But since I only pop elements off the stack by decrementing **vm.sp** (without explicitly setting them to nil), this is also where I can find the elements that were previously on top of the stack. With **LastPoppedStackElem**, I can make change VM tests to make sure **OpPop** is actually handled correctly:
```go
// vm/vm_test.go
func runVmTests(t *testing.T, tests []vmTestCase) {
	t.Helper()

	// [...]

		stackElem := vm.LastPoppedStackElem()

		testExpectedObject(t, tt.expected, stackElem)
	}
}
```
And this is enough to break mu tests:
```shell
go test ./vm      
--- FAIL: TestIntegerArithmetic (0.00s)
    vm_test.go:86: testIntegerObject failed: object is not Integer. got=<nil> (<nil>)
    vm_test.go:86: testIntegerObject failed: object is not Integer. got=<nil> (<nil>)
    vm_test.go:86: testIntegerObject failed: object has wrong value. got=2, want=3
FAIL
FAIL    github.com/vit0rr/mumu/vm       0.799s
FAIL
```

In order to get them to pass again, I need to tell the VM to keep its stack clean and tidy:
```go
// vm/vm.go
func (vm *VM) Run() error {
    // [...]
		case code.OpPop:
			vm.pop()
		}
	}

	return nil
}
```
With that, stack hygiene is restored:
```shell
go test ./vm  
ok      github.com/vit0rr/mumu/vm       0.460s
```

And I also need to fix my REPL, where I still use **StackTop**, by replacing it with **LastPoppedStackElem**:
```go
// repl/repl.go
func Start(in io.Reader, out io.Writer, useCompiler bool) {
	// [...]
			lastPopped := machine.LastPoppedStackElem()
			io.WriteString(out, lastPopped.Inspect())
			io.WriteString(out, "\n")

			continue
		}
```

And on main:
```go
// main.go
	if useCompiler {
		// [...]

		lastPopped := machine.LastPoppedStackElem()
		io.WriteString(os.Stdout, lastPopped.Inspect())
		io.WriteString(os.Stdout, "\n")

		return nil
	}
```

That means I can move on and safely do more arithmetic on the stack without the stack slowly blowing up in my face.