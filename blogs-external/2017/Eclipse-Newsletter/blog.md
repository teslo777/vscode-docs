# Language Server Protocol

Authors:
[Dirk Bäumer](https://github.com/dbaeumer),
[Erich Gamma](https://github.com/egamma),
[Sean McBreen](https://github.com/seanmcbreen)

## How it started

The protocol was not created out of thin air, but it builds up on work and experiences from many others. Editors like [Vim](http://www.vim.org/) or [Emacs](https://www.gnu.org/software/emacs/) have used language servers or demons to provide semantic auto complete support since a while.

This concept was picked up by [OmniSharp](http://www.omnisharp.net/). OmniSharp provides auto complete and other rich editing features for C#. Initially OmniSharp used the http protocol with a JSON payload. OmniSharp has been integrated into several [editors](http://www.omnisharp.net/#integrations). One of them being VS Code.

Around the same time Microsoft started the work on a TypeScript language server, with the idea to support TypeScript in editors like [Emacs](https://www.gnu.org/software/emacs/) and [Sublime](https://www.sublimetext.com/). The [TypeScript Server](https://github.com/Microsoft/TypeScript/tree/master/src/server) communicates through stdin/stdout with the server process and uses a JSON payload inspired by the [V8 debugger protocol](https://github.com/v8/v8/wiki/Debugging-Protocol) for requests and responses. This TypeScript server has been integrated into the [TypeScript Sublime plugin](https://github.com/Microsoft/TypeScript-Sublime-Plugin). VS Code also uses this language server for its rich TypeScript editing experience.

After having consumed two different language servers in VS Code, we started to think about a common language server protocol for VS Code. The idea was to simplify the consumption of language server. A common protocol enables to write the integration code into the host once and then to reuse it for other language servers that speak the same protocol.

We took the language protocol of the TypeScript server as a seed and made it more general and language neutral. Later the protocol was enriched with full language features inspired by the [VS Code language API](https://code.visualstudio.com/docs/extensionAPI/vscode-api#_languages). We selected [JSON-RPC](http://www.jsonrpc.org/) as the remote invocation technique due to its simplicity and support in many programming languages.

The first consumers of this protocol were 'linters' that were kept alive and where running as a simple language server that returns problems (warnings and errors) by calling into a linter library. Examples are VS Code's [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint) or [TSLint](https://marketplace.visualstudio.com/items?itemName=eg2.tslint) extension.

The [VS Code PowerShell extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell) was among the first adopters of the full language server protocol outside of the VS Code team.

## How it works

As said, language servers run in their own process and tools like VS Code communicate with them using the language protocol over JSON-RPC. The actual transport channel can either be `stdio`, `sockets`, `named pipes` or `node ipc` if both the client and server is written in node. Here’s an example of how a tool and a language server could communicate semantic information during a routine editing session: 

![language server protocol](language-server-sequence.png)

* The user opens a file (referred to as a *document*) in the tool: The tool notifies the language server that a document is open ('textDocument/didOpen') and that the information about that document is maintained by the tool in memory.

* The user makes edits: The tool notifies the server about the document change ('textDocument/didChange') and the semantic information of the program is updated by the language server. As this happens, the language server analyses this information and notifies the tool with the errors and warnings ('textDocument/publishDiagnostics') that are found.

* The user executes "Goto Definition" on a symbol in the editor: The tool sends a 'textDocument/definition' request with two parameters (1) the document URI and (2) the position where the goto definition request occurred in the editor to the server. The server responds with a Location literal that holds the document URI and the position of the symbol's definition.

* The user closes the document (file): A 'textDocument/didClose' notification is sent from the tool, informing the language server that the document is now no longer in memory and instead maintained by (i.e. stored on) the file system.

This example also shows that the protocol is driven by features an editor or IDE usually provides and the data types used by the protocol are editor or IDE 'data types' like the open text document and the position of the cursor and not programming languages types like abstract syntax trees or compiler symbols (e.g. resolved types, namespaces, ...). Looking at the 'textDocument/definition' request again here is what goes over the wire for a simple C++ example where the user executes 'Goto Definition' in a file use.cpp and the symbol is defined in provide.cpp:

```json
{
	"jsonrpc": "2.0",
	"id" : 1,
	"method": "textDocument/definition",
	"params": {
		"textDocument": {
			"uri": "file:///p%3A/mseng/VSCode/Playgrounds/cpp/use.cpp"
		},
		"position": {
			"line": 3,
			"character": 12
		}
	}
}
```
The response looks like this:
```json
{
	"jsonrpc": "2.0",
	"id": "1",
	"result": {
		"uri": "file:///p%3A/mseng/VSCode/Playgrounds/cpp/provide.cpp",
		"range": {
			"start": {
				"line": 0,
				"character": 4
			},
			"end": {
				"line": 0,
				"character": 11
			}
		}
	}
}
```

Choosing this approach is one of the major success factors of the language server protocol since it is much easier to standardize a text document URI or a cursor position than standardizing an abstract syntax tree and compiler symbols that are valid for any type of programming languages.

VS Code starts multiple language server instances during a programming session one for each programming language to be supported. But the protocol spoken between the server and VS Code is the same. For example to naviagate to a defintion of a symbol a 'textDocument/definiton' request is sent and a 'Location' is returned in a response.

![language server protocol](language-server.png)

Since not every language server might be able to implement all language features and not all clients might expose the same set of features either the language server protocol uses the concept of capabilities to announce the client's and server's feature set. So a server can for example announce that it can handle the 'textDocument/definition' request but it might not handle the 'workspace/symbol' request. Clients for example can flag that they are able to provide notifications before a document is saved so that a server can compute textual edits to format the text document or to auto fix problems in it on save.

## How it evolved

@erich, @sean can you write something about how RedHat and CodeEny approached used 

When RedHat and CodeEny started to use the language server protocol we moved the complete specification of the protocol to a public [GitHub repository](https://github.com/Microsoft/language-server-protocol). As it continues to be adopted by more languages and tools, we intend to support and evolve the protocol, along with partners and others in the open source community. Anyone can ask questions, file issues, or submit pull requests on the repo, just like any other open source project.

Having the protocol public resulted in a broader adoption of the protocol both for language servers supporting more programming languages as well as clients implementing the protocol to make use of the server ecosystem. On the client side the protocol is now supported by [VS Code](https://code.visualstudio.com/), [MS Monaco Editor](https://www.npmjs.com/package/monaco-languageclient), [Eclipse Che](https://github.com/eclipse/che/issues/1287), [Eclipse IDE](https://projects.eclipse.org/projects/technology.lsp4e), [Emacs]((https://www.gnu.org/software/emacs/)), [GNOME Builder](https://git.gnome.org/browse/gnome-builder/tree/libide/langserv) and [Vim](https://github.com/autozimu/LanguageClient-neovim). An extensive list of the language servers available can be found [here](https://github.com/Microsoft/language-server-protocol/wiki/Protocol-Implementations). 

In July 2016 Microsoft hosted a hackathon in its Zurich lab to work on a Java Language server that can be used in EclipseChe, Orion and VS Code. We made some great progress in one week and it was good to see our old friends from IBM again.

![Java language server hackathon](java-server-hackathon.png)

The VS Code team also maintains an [npm module](https://www.npmjs.com/package/vscode-languageclient) to ease the integration of a language server into a VS Code extension and another [npm module](https://www.npmjs.com/package/vscode-languageserver) to write a language server using node.

## Summary

Still to be written. Something like: the protocol was originally VS Code specific. Great to see how the OS community picked it up, improved it and adopted it in other clients and developed a server ecosystem around it.