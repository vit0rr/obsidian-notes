My. compiler needs to: walk the AST recursively, find `*ast.IntegerLiterals`, evaluate them and turn them into `*object.Integers`, add those to the **constants** field, and add **OpConstant**instructions to t is internal **instructions** slice.

### Walking the AST.
That's something I already did in the **Eval** function I wrote in the previous book and there is no reason to change the approach. How I get `*ast.IntegerLiterals`:

```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	case *ast.Program:
		for _, s := range node.Statements {
			err := c.Compile(s)
			if err != nil {
				return err
			}
		}

	case *ast.ExpressionStatement:
		err := c.Compile(node.Expression)
		if err != nil {
			return err
		}

	case *ast.InfixExpression:
		err := c.Compile(node.Left)
		if err != nil {
			return err
		}

		err = c.Compile(node.Right)
		if err != nil {
			return err
		}

	case *ast.IntegerLiteral:
		// what's  now??
	}

	return nil
}
```

I go first through all the **node.Statements** in `*ast.Program` and call `c.Compile` with each of them. That gets on level deeper in the AST, where I find an `*ast.ExpressionStatement`, and that's that represents the **1 + 2** in my test. Then, I compile the `node.Expression`of that `*ast.ExpressionStatement` and end up with an `*ast.InfixExpression` of which we have to compile the node left and right sides.

And about the TODO, I need. to evaluate them (the IntegerLiterals). That's safe to do because. literals are constant expressions and their value does not change. A **2** will always evaluate to **2**. And evaluate means "creating an `*object.Integer`.

```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	// ...

	case *ast.IntegerLiteral:
		integer := &object.Integer{Value: node.Value}
	}

	// ...
}
```

Now I have the result of the evaluation - integer- at hand and can add it to my constant pool. For this, I'll create another helper to the compiler, called `addConstant`.

```go
// compiler/compiler.go
func (c *Compiler) addConstant(obj object.Object) int {
	c.constants = append(c.constants, obj)
	return len(c.constants) - 1
}
```

I append the **obj** to the end of the compilers *constants* slice and give it its very own identifier by returning its index in the *constants* slide. This identifier will now be used as the operand for the **OpConstant** instruction that should cause the **VM** to load this constant from the **constants** pool on to the stack. It means that now, I'm able to add constants and to remember their identifiers.

Let's emit (generate/output) a first instruction. Or, in other words, generate and instruction and add it to the results, either by printing it, writing it to a file or by adding it to a collection in memory (that last one is what I'll going to do).

```go
// compiler/compiler.go
func (c *Compiler) addConstant(obj object.Object) int {
	c.constants = append(c.constants, obj)
	return len(c.constants) - 1
}

func (c *Compiler) emit(op code.Opcode, operands ...int) int {
	ins := code.Make(op, operands...)
	pos := c.addInstruction(ins)

	return pos
}
```

**emit** returns the starting position of the just-emitted instruction. I'll use this return value later on when I need to go back in **c.instructions** and modify it...

In the **Compile** method I can now use **addConstant** and **emit** to make one delicate change:

```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	// ...

	case *ast.IntegerLiteral:
		integer := &object.Integer{Value: node.Value}
		c.emit(code.OpConstant, c.addConstant(integer))
	}

	// ...
}
```    

And by now, compiler tests pass!
```shell
➜ go test ./compiler 
ok      github.com/vit0rr/mumu/compiler 0.486s
```

Now, I wanna recap what compiler are doing by now. I have one opcode defined, **OpConstant**. I have a tiny compiler that knows how to walk an AST and emit such an **OpConstant** instruction. The compiler also knows how to evaluate constant integer literal expressions and how to add them to its constant pool. And the compiler's interface allow me to pass around the result of the compilation, including the emitted instructions and the constant pool.