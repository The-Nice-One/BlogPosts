# Enhancing Julia's REPL

"How do you enhance Julia whilst keeping performance unharmed?"

# Introduction

As a programmer who loves the Julia programming language for its multiple dispatch design, and especially for its built-in REPL. I have always wanted to see more features in the Julia REPL. Whilst it features auto-completion for Julia syntax, a simple history list to recall expressions evaluated, and different modes such as help mode for documentation and Pkg mode for managing your Julia project. I've always wanted more functionality. Maybe for aesthetics, but nevertheless for enchantments.

# InteractiveErrors.jl & OhMyREPL.jl
Luckily, I am not the only one who wishes to have an enhanced REPL. Packages such as [OhMyREPL.jl](https://github.com/KristofferC/OhMyREPL.jl) exist. This package adds syntax highlighting, automatic bracket completion and coloring, along with other cool features inside the Julia REPL. Additionally, the [InteractiveErrors.jl](https://github.com/MichaelHatherly/InteractiveErrors.jl) package makes errors easier to understand by providing an interactive error message. Allowing you to view precisely at which lines your code failed, and providing tools to debug faster, like copying the stacktrace. So let's begin installing the package!

First, start a Julia REPL session and enter Pkg mode by pressing the "]" key. Then, we can add the packages we want to install with the following.
```bash
add InteractiveErrors OhMyREPL
```
Note, we install these packages in our global environment so that we can add setup code within the startup file. More on that later.

In this blog, we will create a custom REPL theme for Julia syntax. For this, we will also need the [Crayons.jl](https://github.com/KristofferC/Crayons.jl) package, which provides a common terminal colors interface for Julia.
```bash
add Crayons
```

# Editing the startup.jl
[startup.jl](https://docs.julialang.org/en/v1/manual/command-line-interface/#Startup-file) is a special file located in the following location, `~/.julia/config/startup.jl`. This code in this file is automatically executed when starting a Julia REPL session. However, this can be overridden by running Julia with the `--startup-file=no` flag. The `startup.jl` is quite valuable and allows us to load any functionality into each REPL session, such as the OhMyREPL.jl and InteractiveErrors.jl packages we've added. Additionally, I've included some extra goodies such as a modern Julia Banner. Let's get started editing the `startup.jl` by adding the following line to import the Pkg package, which will be used throughout the startup.
```julia
using Pkg; Pkg
```
The first thing we'll get working is syntax highlighting. For this, I've created a custom theme that uses your terminal colors. You can learn more about changing the syntax colors in the [OhMyREPL.jl documentation](https://kristofferc.github.io/OhMyREPL.jl/latest/).
```julia
function create_simple_colors_colorscheme()
    # Defines a new ColorScheme which we can edit
    scheme = SyntaxHighlighter.ColorScheme()
    # Change the colors of each indentifier for the ColorScheme
    SyntaxHighlighter.symbol!(scheme, Crayon(foreground = :magenta))
    SyntaxHighlighter.comment!(scheme, Crayon(foreground = :dark_gray))
    SyntaxHighlighter.string!(scheme, Crayon(foreground = :blue))
    SyntaxHighlighter.call!(scheme, Crayon(foreground = :blue))
    SyntaxHighlighter.op!(scheme, Crayon(foreground = :yellow))
    SyntaxHighlighter.keyword!(scheme, Crayon(foreground = :magenta))
    SyntaxHighlighter.macro!(scheme, Crayon(foreground = :magenta))
    SyntaxHighlighter.function_def!(scheme, Crayon(foreground = :yellow))
    SyntaxHighlighter.text!(scheme, Crayon(foreground = :default))
    SyntaxHighlighter.error!(scheme, Crayon(foreground = :red))
    SyntaxHighlighter.argdef!(scheme, Crayon(foreground = :magenta))
    SyntaxHighlighter.number!(scheme, Crayon(foreground = :blue))
    # Adds the ColorScheme into the list of themes
    SyntaxHighlighter.add!("SimpleColors", scheme)
end
```

Another addition I've included is a custom Julia `banner()` function adapted from [BetterBanner.jl](https://github.com/longemen3000/BetterBanner.jl) to support modern Julia versions.
```julia
function banner(io = stdout)
    GIT_VERSION_INFO = Base.GIT_VERSION_INFO
    if GIT_VERSION_INFO.tagged_commit
        commit_string = Base.TAGGED_RELEASE_BANNER
    elseif isempty(GIT_VERSION_INFO.commit)
        commit_string = ""
    else
        days = Int(floor((ccall(:jl_clock_now, Float64, ()) - GIT_VERSION_INFO.fork_master_timestamp) / (60 * 60 * 24)))
        days = max(0, days)
        unit = days == 1 ? "day" : "days"
        distance = GIT_VERSION_INFO.fork_master_distance
        commit = GIT_VERSION_INFO.commit_short

        if distance == 0
            commit_string = "Commit $(commit) ($(days) $(unit) old master)"
        else
            branch = GIT_VERSION_INFO.branch
            commit_string = "$(branch)/$(commit) (fork: $(distance) commits, $(days) $(unit))"
        end
    end
    empty_str = ""
    logoplusspace = 26
    commit_date = isempty(Base.GIT_VERSION_INFO.date_string) ? "" : "($(split(Base.GIT_VERSION_INFO.date_string)[1]))"
    doc = "Documentation:"
    doc_link = "https://docs.julialang.org"
    general_help = """Type "?" for help"""
    pkg_help = """"]?" for Pkg help."""
    _,width = displaysize(io)
    if width >= 67
        t1 = empty_str
        t2 = doc * " " * doc_link
        t3 = empty_str
        t4 = general_help * ", " * pkg_help
        t5 = empty_str
        t6 = "Version $(Base.VERSION) $(commit_date)"
        t7 = commit_string
        t8 = empty_str
        extra = """

        """
    elseif 44 <= width <= 67
        t1 = doc
        if width >= 52
            t2 = doc_link
        else
            t2 = "docs.julialang.org"
        end
        t3 = empty_str
        t4 = general_help * ","
        t5 = pkg_help
        t6 = empty_str
        version_str = "Ver. $(Base.VERSION) $commit_date"
        if length(version_str) + logoplusspace < width
            t7 = version_str
            t8 = empty_str
        else
            t7 = "Ver. $(Base.VERSION)"
            t8 = commit_date
        end
        extra = """

        $commit_string

        """
    else
        t1 = empty_str
        t2 = empty_str
        t3 = empty_str
        t4 = empty_str
        t5 = empty_str
        t6 = empty_str
        t7 = empty_str
        t8 = empty_str
        #expect overflowing, but the logo is unaffected
        extra = """

        $doc
        $doc_link

        $(general_help), $pkg_help

        Ver. $(Base.VERSION) $commit_date

        $commit_string

        """
    end

    separator = ifelse(44 <= width, " | ",empty_str)
    tt1 = separator * t1
    tt2 = separator * t2
    tt3 = separator * t3
    tt4 = separator * t4
    tt5 = separator * t5
    tt6 = separator * t6
    tt7 = separator * t7
    tt8 = separator * t8
    if get(io, :color, false)
        c = Base.text_colors
        tx = c[:normal] # text
        jl = c[:normal] # julia
        d1 = c[:bold] * c[:blue]    # first dot
        d2 = c[:bold] * c[:red]     # second dot
        d3 = c[:bold] * c[:green]   # third dot
        d4 = c[:bold] * c[:magenta] # fourth dot



    print("""

                    $(d3)⢰⣿⣿⡆$(tx)   $(tt1)
                    $(d3)⠈⠛⠛⠁$(tx)   $(tt2)
     $(d1)⢰⣿⣿⡆$(jl)       ⣴⣾$(d2)⢰⣿⣿⡆$(d4)⢰⣿⣿⡆$(tx) $(tt3)
     $(d1)⠈⠛⠛⠁$(jl)       ⣿⣿$(d2)⠈⠛⠛⠁$(d4)⠈⠛⠛⠁$(tx) $(tt4)
      ⣿⣿ ⣿⣿  ⣿⣿ ⣿⣿ ⣴⣾ ⣾⠟⠛⣿⣷$(tt5)
      ⣿⣿ ⣿⣿  ⣿⣿ ⣿⣿ ⣿⣿⢀⣴⡾⠋⣿⣿$(tt6)
      ⣿⣿ ⠙⢿⣶⠞⣿⣿ ⣿⣿ ⣿⣿⠈⠻⣷⣶⡿⣿$(tt7)
    ⢶⣦⣿⠟                   $(tt8)
    $(extra)""")
    else
        print("""
                    ⢰⠉⠉⡆   $(tt1)
                    ⠈⠒⠒⠁   $(tt2)
     ⢰⠉⠉⡆       ⣴⣾⢰⠉⠉⡆⢰⠉⠉⡆ $(tt3)
     ⠈⠒⠒⠁       ⣿⣿⠈⠒⠒⠁⠈⠒⠒⠁ $(tt4)
      ⣿⣿ ⣿⣿  ⣿⣿ ⣿⣿ ⣴⣾ ⣾⠟⠛⣿⣷$(tt5)
      ⣿⣿ ⣿⣿  ⣿⣿ ⣿⣿ ⣿⣿⢀⣴⡾⠋⣿⣿$(tt6)
      ⣿⣿ ⠙⢿⣶⠞⣿⣿ ⣿⣿ ⣿⣿⠈⠻⣷⣶⡿⣿$(tt7)
    ⢶⣦⣿⠟                   $(tt8)
    $(extra)""")
    end
end
```
And now we simply define the `atreplinit()` function, which is executed when the REPL begins. This is where we initialize the packages and use the functions we've defined above.
```julia
atreplinit() do repl
    try
        # Brings namespaces into scope.
        @eval using OhMyREPL
       	@eval using Crayons
       	@eval import OhMyREPL: Passes.SyntaxHighlighter
       	# Allows autocompletion of brackets. For example, if you type "(" in the REPL, ")" will be added automatically.
       	@eval enable_autocomplete_brackets(true)
       	# This is a personal addition. I've made the standard "julia>" input prompt in the REPL feel like the input prompt in Pkg mode of the REPL.
       	@eval OhMyREPL.input_prompt!("(@v$(Base.VERSION)) julia>", :green)
       	# Makes each set of nested brackets have a separate color for better distinction.
        @eval OhMyREPL.Passes.RainbowBrackets.activate_16colors()
        # Calls our custom create_`simple_colors_colorscheme()` to create the "SimpleColors" theme.
       	@eval create_simple_colors_colorscheme()
       	# Sets the color theme of OhMyREPL's SyntaxHighlighter to our "SimpleColors" theme.
       	@eval colorscheme!("SimpleColors")

        # Enables interactive runtime errors. Unlike OhMyREPL, no configuration is needed; simply use the package.
       	@eval using InteractiveErrors

       	# Calls our custom Julia banner defined in the previous steps.
       	@eval banner()
    catch e
        # If our startup failed, just report the error in the REPL session.
        @warn e
    end
end
```

# SysImages with PackageCompiler.jl
All the new enhancements we've added come at a cost for the Julia REPL. The packages add overhead time when starting the REPL, as Julia needs to run our `startup.jl` file, which loads packages and executes code.

Additionally, due to Julia's AOT precompilation, the first time we type in the REPL, Julia needs to compile the syntax highlighting code, auto-completion code, and more. This creates lag spikes the first time certain features in the REPL are used.

In response, we can create a system image for Julia, including all the packages and methods compiled. Then we pass the image as an argument when we start the Julia REPL. In fact, Julia comes with an image containing all the Base module methods and several other features, and is used by default.

To compile our own image, we will use a Julia package called [PackageCompiler.jl](https://github.com/JuliaLang/PackageCompiler.jl). This package may also be used to compile our Julia code into a C library or even a distributable binary.

But before we compile our Image, we need to know exactly which functions we need to compile. In our case, we need all of Julia's default image functionality, the OhMyREPL.jl, InteractiveErrors.jl, Crayons.jl packages, and all the functions that get called upon REPL startup. Knowing all this would be a tedious task. As a result, Julia provides the `--trace-compile` flag, which writes all the functions executed in the Julia REPL to a file we specify. In a terminal session, we can run the following command to start the Julia REPL with compile tracebacks.
```bash
julia --trace-compile=precompile_script.jl
```
Now, within the REPL in evaluation mode (default mode), we can type statements such as the following.
```julia
function hello(name::String)
    println("Hello, " * name * "!")
end
```
As you type, you may experience lag spikes. This is great, and means Julia is precompiling syntax highlighting code along with related functionality. We can also cause a runtime error with the following.
```julia
numbers = []
numbers[1]
```
Now, an interactive error like the following should pop up and precompile.
```bash
BoundsError: attempt to access 0-element Vector{Any} at index [1]
     (stacktrace)
      (user)
 >     Base
   +    throw_boundserror ./essentials.jl:15
   +   [inlined]
   +   Base
   +   [top-level]
   +  (system)
```
Awesome! Most, if not all, functionality should have been precompiled by now. Let's return to shell mode by pressing the ";" key, or go back to our terminal by typing the following.
```julia
exit()
```
And we can view the `./precompile_script.jl` file in a visual editor or via the `cat` command.
```bash
cat precompile_script.jl
```
Which should display a handful of `precompile()` function calls similar to the following, in my case.
```julia
precompile(Tuple{typeof(Base.vect), Array{String, 1}, Vararg{Array{String, 1}}})
precompile(Tuple{typeof(Base.iterate), Array{Array{String, 1}, 1}})
precompile(Tuple{typeof(Base.print_to_string), String, String, String, String, String, String, String, String, String, String, Vararg{Any}})
precompile(Tuple{REPLExt.var"#__init__##0#__init__##1", REPL.LineEditREPL})
precompile(Tuple{typeof(Base.setindex!), Base.Dict{Any, Any}, Any, Char})
precompile(Tuple{Main.var"#2#3", REPL.LineEditREPL})
precompile(Tuple{typeof(Artifacts.__artifact_str), Module, String, Base.SubString{String}, String, Base.Dict{String, Any}, Base.SHA1, Base.BinaryPlatforms.Platform, Base.Val{Artifacts}})
precompile(Tuple{typeof(JLLWrappers.get_julia_libpaths)})
precompile(Tuple{typeof(Base.getproperty), REPL.LineEditREPL, Symbol})
precompile(Tuple{typeof(Base.getindex), Array{REPL.LineEdit.TextInterface, 1}, Int64})
precompile(Tuple{typeof(Base.findfirst), Function, Array{REPL.LineEdit.TextInterface, 1}})
precompile(Tuple{typeof(Base.findnext), Base.Fix{2, typeof(isa), Type{REPL.LineEdit.PrefixHistoryPrompt}}, Array{REPL.LineEdit.TextInterface, 1}, Int64})
precompile(Tuple{typeof(Base.getindex), Type{Base.Dict{Any, Any}}, Base.Dict{Any, Any}, Base.Dict{Char, Any}})
precompile(Tuple{typeof(OhMyREPL.BracketInserter.enable_autocomplete_brackets), Bool})
precompile(Tuple{typeof(OhMyREPL.input_prompt!), String, Symbol})
precompile(Tuple{typeof(OhMyREPL.update_interface), REPL.LineEdit.ModalInterface})
precompile(Tuple{typeof(Base.setproperty!), REPL.LineEdit.Prompt, Symbol, String})
precompile(Tuple{typeof(OhMyREPL.Passes.RainbowBrackets.activate_16colors)})
precompile(Tuple{typeof(OhMyREPL.colorscheme!), String})
precompile(Tuple{typeof(Base.getproperty), OhMyREPL.Passes.SyntaxHighlighter.SyntaxHighlighterSettings, Symbol})
precompile(Tuple{typeof(Base.vect), Base.Dict{String, Any}, Vararg{Any}})
precompile(Tuple{typeof(Base.:(*)), Int64, Int64, Int64})
precompile(Tuple{typeof(Base.print), Base.TTY, String})
precompile(Tuple{typeof(Base.iszero), Float64})
precompile(Tuple{OhMyREPL.var"#__init__##0#__init__##1"})
precompile(Tuple{InteractiveErrors.var"#setup_repl##0#setup_repl##1"})
precompile(Tuple{typeof(Base.getproperty), REPL.REPLBackend, Symbol})
precompile(Tuple{Base.var"#readcb_specialized#uv_readcb##0", Base.TTY, Int64, UInt64})
precompile(Tuple{typeof(Base.peek), Base.TTY, Type{UInt8}})
precompile(Tuple{typeof(Base.get), Base.Dict{Char, Any}, Char, Nothing})
precompile(Tuple{REPL.LineEdit.var"#match_input##0#match_input##1"{OhMyREPL.Prompt.var"#create_keybindings##2#create_keybindings##3", String}, Any, Any})
precompile(Tuple{OhMyREPL.Prompt.var"#create_keybindings##2#create_keybindings##3", Any, Any, Any})
precompile(Tuple{Type{Base.IOContext{IO_t} where IO_t<:IO}, Base.GenericIOBuffer{Memory{UInt8}}, Base.TTY})
precompile(Tuple{typeof(Base.write), Base.IOContext{Base.GenericIOBuffer{Memory{UInt8}}}, String})
precompile(Tuple{typeof(Base.Unicode.textwidth), String})
precompile(Tuple{typeof(Base.get), Base.TTY, Symbol, Bool}) # recompile
precompile(Tuple{typeof(Base.unsafe_write), Base.IOContext{Base.GenericIOBuffer{Memory{UInt8}}}, Ptr{UInt8}, UInt64})
precompile(Tuple{typeof(OhMyREPL.untokenize_with_ANSI), Base.IOContext{Base.GenericIOBuffer{Memory{UInt8}}}, OhMyREPL.PassHandler, Array{JuliaSyntax.Token, 1}, String, Int64})
precompile(Tuple{typeof(Base.getproperty), REPL.LineEdit.ModeState, Symbol})
precompile(Tuple{typeof(Base.position), Base.GenericIOBuffer{Memory{UInt8}}})
precompile(Tuple{typeof(Base.seek), Base.GenericIOBuffer{Memory{UInt8}}, Int64})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:keep,), Tuple{Bool}}, typeof(Base.readline), Base.GenericIOBuffer{Memory{UInt8}}})
precompile(Tuple{typeof(Base.divrem), Int64, Int64})
precompile(Tuple{REPL.LineEdit.var"#match_input##0#match_input##1"{OhMyREPL.Prompt.var"#create_keybindings##42#create_keybindings##43", String}, Any, Any})
precompile(Tuple{typeof(Base.getproperty), OhMyREPL.PassHandler, Symbol})
precompile(Tuple{OhMyREPL.Prompt.var"#create_keybindings##42#create_keybindings##43", Any, Any, Any})
precompile(Tuple{Base.Returns{Symbol}, Any})
precompile(Tuple{typeof(Base.indexed_iterate), Tuple{Base.GenericIOBuffer{Memory{UInt8}}, Bool, Bool}, Int64})
precompile(Tuple{typeof(Base.indexed_iterate), Tuple{Base.GenericIOBuffer{Memory{UInt8}}, Bool, Bool}, Int64, Int64})
precompile(Tuple{REPL.var"#do_respond#74"{Bool, Bool, REPL.var"#setup_interface##12#setup_interface##13"{REPL.LineEditREPL, REPL.REPLHistoryProvider}, REPL.LineEditREPL, REPL.LineEdit.Prompt}, REPL.LineEdit.MIState, Any, Bool}) # recompile
precompile(Tuple{typeof(Base.put!), Base.Channel{Any}, Tuple{Expr, Int64}})
precompile(Tuple{typeof(InteractiveErrors.wrap_errors), Expr})
precompile(Tuple{typeof(Base.getindex), Base.RefValue{Ptr{UInt8}}})
precompile(Tuple{typeof(Base.fl_lower), Expr, Module, Ptr{UInt8}, UInt64})
precompile(Tuple{typeof(Base.:(var"==")), Type, GlobalRef})
precompile(Tuple{typeof(Base.:(var"==")), Function, GlobalRef})
precompile(Tuple{typeof(Base.unsafe_convert), Type{Ptr{Ptr{UInt8}}}, Base.RefValue{Ptr{UInt8}}})
precompile(Tuple{Type{Base.IOContext{IO_t} where IO_t<:IO}, Base.TTY, Pair{Symbol, Array{Tuple{String, Int64}, 1}}})
precompile(Tuple{Type{Base.IOContext{IO_t} where IO_t<:IO}, Base.IOContext{Base.TTY}, Pair{Symbol, Module}})
precompile(Tuple{typeof(Base.indexed_iterate), Pair{Any, Bool}, Int64})
precompile(Tuple{typeof(Base.indexed_iterate), Pair{Any, Bool}, Int64, Int64})
precompile(Tuple{typeof(Base.Multimedia.display), Any}) # recompile
precompile(Tuple{Type{Base.BottomRF{T} where T}, Type{Base.IOContext{IO_t} where IO_t<:IO}})
precompile(Tuple{OhMyREPL.var"#5#6"{REPL.REPLDisplay{REPL.LineEditREPL}, Base.Multimedia.MIME{:var"text/plain"}, Base.RefValue{Any}}, Base.IOContext{Base.TTY}})
precompile(Tuple{typeof(Base.show), Base.IOContext{Base.TTY}, Base.Multimedia.MIME{:var"text/plain"}, Int64})
precompile(Tuple{typeof(Base.println), Base.IOContext{Base.TTY}})
precompile(Tuple{REPL.LineEdit.var"#match_input##0#match_input##1"{OhMyREPL.BracketInserter.var"#insert_into_keymap!##0#insert_into_keymap!##1"{Array{Char, 1}, Char, Char}, String}, Any, Any})
precompile(Tuple{OhMyREPL.BracketInserter.var"#insert_into_keymap!##0#insert_into_keymap!##1"{Array{Char, 1}, Char, Char}, REPL.LineEdit.MIState, REPL.LineEditREPL, Vararg{Any}})
precompile(Tuple{REPL.LineEdit.var"#match_input##0#match_input##1"{OhMyREPL.Prompt.var"#create_keybindings##26#create_keybindings##27", String}, Any, Any})
precompile(Tuple{OhMyREPL.Prompt.var"#create_keybindings##26#create_keybindings##27", Any, Any, Any})
precompile(Tuple{typeof(REPL.Terminals.cmove_up), REPL.Terminals.TerminalBuffer})
precompile(Tuple{REPL.LineEdit.var"#match_input##0#match_input##1"{OhMyREPL.BracketInserter.var"#insert_into_keymap!##10#insert_into_keymap!##11"{Array{Char, 1}, Array{Char, 1}}, String}, Any, Any})
precompile(Tuple{OhMyREPL.BracketInserter.var"#insert_into_keymap!##10#insert_into_keymap!##11"{Array{Char, 1}, Array{Char, 1}}, REPL.LineEdit.MIState, REPL.LineEditREPL, String})
precompile(Tuple{typeof(REPL.add_locals!), Any, Symbol}) # recompile
precompile(Tuple{typeof(Base.show), Base.IOContext{Base.TTY}, Base.Multimedia.MIME{:var"text/plain"}, Function})
precompile(Tuple{typeof(Base.print), Base.IOContext{Base.TTY}, Vararg{String, 4}})
precompile(Tuple{Type{InteractiveErrors.CapturedError}, UndefVarError, Array{Union{Ptr{Nothing}, Base.InterpreterIP}, 1}})
precompile(Tuple{Type{NamedTuple{(:cursor,), T} where T<:Tuple}, Tuple{Int64}})
precompile(Tuple{typeof(Base.pairs), NamedTuple{(:cursor,), Tuple{Int64}}})
precompile(Tuple{typeof(Base.merge), NamedTuple{(), Tuple{}}, Base.Pairs{Symbol, Int64, Tuple{Symbol}, NamedTuple{(:cursor,), Tuple{Int64}}}})
precompile(Tuple{Type{Pair{A, B} where B where A}, String, Function})
precompile(Tuple{typeof(InteractiveErrors.explore), InteractiveErrors.CapturedError})
precompile(Tuple{typeof(InteractiveErrors.explore), Base.TTY, InteractiveErrors.CapturedError})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:context,), Tuple{Pair{Symbol, Bool}}}, typeof(Base.sprint), Function, UndefVarError})
precompile(Tuple{Base.var"##sprint#442", Pair{Symbol, Bool}, Int64, typeof(Base.sprint), typeof(Base.showerror), UndefVarError})
precompile(Tuple{typeof(Base.UndefVarError_hint), Base.IOContext{Base.GenericIOBuffer{Memory{UInt8}}}, UndefVarError})
precompile(Tuple{typeof(REPL.UndefVarError_REPL_hint), Base.IOContext{Base.GenericIOBuffer{Memory{UInt8}}}, UndefVarError})
precompile(Tuple{typeof(Base.process_backtrace), Array{Union{Ptr{Nothing}, Base.InterpreterIP}, 1}})
precompile(Tuple{typeof(Base.indexed_iterate), Tuple{Base.StackTraces.StackFrame, Int64}, Int64})
precompile(Tuple{typeof(Base.indexed_iterate), Tuple{Base.StackTraces.StackFrame, Int64}, Int64, Int64})
precompile(Tuple{Type{InteractiveErrors.StackFrameWrapper}, Tuple{Base.StackTraces.StackFrame, Int64}}) # recompile
precompile(Tuple{InteractiveErrors.var"#make_nodes#24"{InteractiveErrors.var"#make_nodes#9#25"}, FoldingTrees.Node{Any}, Array{InteractiveErrors.StackFrameWrapper, 1}}) # recompile
precompile(Tuple{typeof(Base.getindex), Array{String, 1}, Base.UnitRange{Int64}})
precompile(Tuple{typeof(Base.Broadcast.materialize), Base.Broadcast.Broadcasted{Base.Broadcast.DefaultArrayStyle{1}, Nothing, typeof(InteractiveErrors.highlight), Tuple{Array{String, 1}}}}) # recompile
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:fold,), Tuple{Bool}}, InteractiveErrors.var"#make_nodes#24"{InteractiveErrors.var"#make_nodes#9#25"}, FoldingTrees.Node{Any}, Array{InteractiveErrors.StackFrameWrapper, 1}}) # recompile
precompile(Tuple{typeof(Base.:(var"==")), Module, Module})
precompile(Tuple{typeof(Base.:(var"==")), Symbol, Module})
precompile(Tuple{typeof(Base.string), Module})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:limit, :keepempty), Tuple{Int64, Bool}}, typeof(Base.eachsplit), String, String})
precompile(Tuple{Base.var"##eachsplit#425", Int64, Bool, typeof(Base.eachsplit), String, String})
precompile(Tuple{Type{Base.SplitIterator{S, F} where F where S<:AbstractString}, String, String, Int64, Bool})
precompile(Tuple{typeof(Base.getproperty), Base.SplitIterator{String, String}, Symbol})
precompile(Tuple{typeof(Base.between), UInt8, UInt8, UInt8})
precompile(Tuple{typeof(Base.reinterpret), Type{Char}, UInt32})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:init,), Tuple{Bool}}, typeof(REPL.TerminalMenus.printmenu), Base.TTY, FoldingTrees.TreeMenu{FoldingTrees.Node{Any}}, Int64})
precompile(Tuple{typeof(REPL.TerminalMenus.readkey), Base.TTY})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:oldstate,), Tuple{Int64}}, typeof(REPL.TerminalMenus.printmenu), Base.TTY, FoldingTrees.TreeMenu{FoldingTrees.Node{Any}}, Int64})
precompile(Tuple{typeof(FoldingTrees.writeoption), Base.GenericIOBuffer{Memory{UInt8}}, InteractiveErrors.StackFrameWrapper, Int64})
precompile(Tuple{typeof(Base.merge), NamedTuple{(), Tuple{}}, NamedTuple{(:bold,), Tuple{Bool}}})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:bold,), Tuple{Bool}}, typeof(InteractiveErrors.style), Symbol})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, typeof(InteractiveErrors.style), Int64})
precompile(Tuple{typeof(Base.println), Base.TTY})
precompile(Tuple{typeof(Base.get), NamedTuple{(:function_name, :directory, :filename, :line_number, :user_stack, :system_stack, :stdlib_module, :base_module, :core_module, :package_module, :unknown_module, :inlined_frames, :toplevel_frames, :repeated_frames, :file_contents, :signature, :source, :line_range, :charset), Tuple{NamedTuple{(:bold,), Tuple{Bool}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color, :bold), Tuple{Symbol, Bool}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color,), Tuple{Symbol}}, NamedTuple{(:color, :format, :highlight), Tuple{Symbol, Bool, Bool}}, NamedTuple{(:color, :bold, :highlight), Tuple{Symbol, Bool, Bool}}, NamedTuple{(:before, :after), Tuple{Int64, Int64}}, Symbol}}, Symbol, Symbol})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:charset,), Tuple{Symbol}}, Type{REPL.TerminalMenus.MultiSelectMenu{C} where C}, Array{String, 1}})
precompile(Tuple{typeof(Base._nthbyte), Base.CodeUnits{UInt8, String}, Int64})
precompile(Tuple{typeof(Base.isequal), UInt8})
precompile(Tuple{typeof(Base.lastindex), Base.CodeUnits{UInt8, String}})
precompile(Tuple{typeof(Base.cconvert), Type{Ptr{UInt8}}, Base.CodeUnits{UInt8, String}})
precompile(Tuple{typeof(Base.getproperty), Base.CodeUnits{UInt8, String}, Symbol})
precompile(Tuple{typeof(Base.last_byteindex), Base.CodeUnits{UInt8, String}})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:init,), Tuple{Bool}}, typeof(REPL.TerminalMenus.printmenu), Base.TTY, REPL.TerminalMenus.MultiSelectMenu{REPL.TerminalMenus.MultiSelectConfig}, Int64})
precompile(Tuple{typeof(Core.kwcall), NamedTuple{(:oldstate,), Tuple{Int64}}, typeof(REPL.TerminalMenus.printmenu), Base.TTY, REPL.TerminalMenus.MultiSelectMenu{REPL.TerminalMenus.MultiSelectConfig}, Int64})
precompile(Tuple{typeof(Base._atexit), Int32}) # recompile
```
In my `precompile_script.jl` I removed the following three lines which may not be in the same order.
```julia
# Don't precompile these lines in order to allow flexibility.
# Allows us to update our theme in the "startup.jl" file when needed.
precompile(Tuple{typeof(Main.create_simple_colors_colorscheme)})
# The default Julia banner does not need to be precompiled since we have our custom implementation.
precompile(Tuple{typeof(Main.banner)})
# Allows us to edit our custom `banner()` function if needed later on.
precompile(Tuple{typeof(Main.banner), Base.TTY})
```
Now that we have our `precompile_script.jl` all written, and we know what to compile. We can proceed with using PackageCompiler.jl. Let's go back to our Julia session and enter Pkg mode to add the PackageCompiler.jl package.
```bash
add PackageCompiler
```
And now, return to evaluation mode by pressing the "backspace" key, and type the following.
```julia
using PackageCompiler
# Note, the first argument (["Crayons", "OhMyREPL", "InteractiveErrors"]) is a list of packages we want to include in our custom image.
PackageCompiler.create_sysimage(["Crayons", "OhMyREPL", "InteractiveErrors"]; sysimage_path="./julia_enhanced.so", precompile_statements_file="precompile_script.jl", incremental=true)
```
This will begin the precompilation process, and your device should have a substantial amount of RAM. Otherwise, you might need to create or increase swap space. In my case, my machine had 6GB of RAM, and I additionally created a swap file of 8GB by following the [Arch Linux Swap guide](https://wiki.archlinux.org/title/Swap). In practice, if your machine has 16GB of RAM, which is fairly common in modern machines, you should be fine.

After a varying amount of time, you should see a completion message in the terminal. In my case, precompilation took a little more than an hour.
```bash
✔ [01h:04m:03s] PackageCompiler: compiling incremental system image
```

Now, if we check the files in our current working directory, we should find a `julia_enhanced.so` file. And we can now test our custom image!

# Testing & Polishing

Exit Julia, and in your terminal session type the following command.
```bash
julia -q -J'/idealy/full/path/to/julia_enhanced.so'
```
The Julia REPL should open as before, except all our enhancements are precompiled! Additionally, you may have noticed that previously, when starting the Julia REPL, two banners were being printed. The `-q` flag silences the default Julia banner, so now we only see our custom Julia banner. But it's a bummer if we need to type that long command every time. So you will likely need to make an alias. This process varies based on your OS. Something like the following should work on Linux or macOS machines using bash and a `~/.bashrc` file.
```bash
alias julia="julia -q -J'/full/path/to/julia_enhanced.so'"
```

# Conclusion

So now you can enhance Julia in many more ways without drastically increasing REPL startup times. And with Julia's [promising list of REPL packages](https://juliahub.com/ui/Search?q=repl&type=packages) found at JuliaHub and the General registry, you can make your perfect REPL.
