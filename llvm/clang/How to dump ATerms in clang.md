# How to dump ATerms in clang

## Background

[ATerms](https://github.com/cwi-swat/aterms) are an exchange format. Currently [clang](https://clang.llvm.org/) supports two formats of dumped ASTs, default and json. 

`clang -cc1 -ast-dump=default [a cpp file]`

`clang -cc1 -ast-dump=json    [a cpp file]`

We would like to have clang dump ATerms directly so that our future research can employ it.

## How clang dumps ASTs

For simplicity, we only investigate how clang dumps the textual format of ASTs (*-ast-dump=default*). A very good starting point is [ClangCheck.cpp](https://github.com/llvm/llvm-project/blob/release/15.x/clang/tools/clang-check/ClangCheck.cpp). The most important function call within it is `FrontendFactory = newFrontendActionFactory(&CheckFactory);` and the most important class is *ClangCheckActionFactory*.s

Under the hood, ClangCheck calls `clang::CreateASTDumper` to create an ASTConsumer. `clang::CreateASTDumper` return an *ASTPrinter* which is responsible for pretty-printing and dumping ASTs. The curcial function is `HandleTranslationUnit()`. 

> HandleTranslationUnit - This method is called when the ASTs for entire translation unit have been parsed.

```cpp
void HandleTranslationUnit(ASTContext &Context) override {
    TranslationUnitDecl *D = Context.getTranslationUnitDecl();

    if (FilterString.empty())
    return print(D);

    TraverseDecl(D);
}
```

Since we don't filter out anything, the control flow goes into `print()`. Then it calls `D->dump(Out, OutputKind == DumpFull, OutputFormat);`. 

let's look at `dump()`. We observe that there are two formats, JSON and default. These correspond to the command line argument *-ast-dump=[format]*. The actual dumping work is delegated to `P.Visit(this)`.
```cpp
LLVM_DUMP_METHOD void Decl::dump(raw_ostream &OS, bool Deserialize,
                                 ASTDumpOutputFormat Format) const {
  ASTContext &Ctx = getASTContext();
  const SourceManager &SM = Ctx.getSourceManager();
 
  if (ADOF_JSON == Format) {
    JSONDumper P(OS, SM, Ctx, Ctx.getPrintingPolicy(),
                 &Ctx.getCommentCommandTraits());
    (void)Deserialize; // FIXME?
    P.Visit(this);
  } else {
    ASTDumper P(OS, Ctx, Ctx.getDiagnostics().getShowColors());
    P.setDeserialize(Deserialize);
    P.Visit(this);
  }
}
```

## How *ASTDumper* works

The class *ASTDumper* inherits *ASTNodeTraverser* and has a member variable *NodeDumper* of type *TextNodeDumper*.  

> ASTNodeTraverser traverses the Clang AST for dumping purposes.

> TextNodeDumper dumps the Clang AST in the text format. 

The class *ASTNodeTraverser* is awesome. Why? Because it eliminates the need to write a visitor ourselves. The only thing left is to control what to dump in the dumper (TextNodeDumper in this case).

## How *TextNodeDumper* works

### *TextNodeDumper.h*

#### *TextTreeStructure*

*TextTreeStructure* is to maintain the tree structure of dumped text. The behavior is controlled by an overloaded template function `AddChild()`. Assume we have indentation already, `Fn DoAddChild` is called to generate the text.

#### *TextNodeDumper*

*TextNodeDumper* declares all functions to specify the dumped context. 

#### How to dump ATerms

To be continued

## Reference

1. [clang::tooling::ClangTool Class Reference](https://clang.llvm.org/doxygen/classclang_1_1tooling_1_1ClangTool.html)
2. [clang::ASTNodeTraverser< Derived, NodeDelegateType > Class Template Reference](https://clang.llvm.org/doxygen/classclang_1_1ASTNodeTraverser.html)
3. [clang::TextNodeDumper Class Reference](https://clang.llvm.org/doxygen/classclang_1_1TextNodeDumper.html)