# Skitter's activity as a version-49 class file, generated from a small DSL

**Status: for decision.** Nothing here is built. Every claim about d8, the JVM and the toolchain was run on
2026-10-19 (JDK 27, d8 9.0.3-dev from build-tools 36.1.0, android-36); the probe is in section 2.

## 1. What exists today

`sysl-lang/skitter-app/activity/src/main/scala/sh/sysl/skitter/SkitterActivity.scala` is **190 lines, about 110 of
them code**, and it is Scala 3. `skitter-app/app/build.gradle.kts` runs a task, `compileActivity`, that execs
`sbt -batch package` in `activity/` (classpath handed over in `SKITTER_CLASSPATH`: `android.jar` plus the
`classes.jar` inside each SDL AAR), copies the jar to `activity/target/skitter-activity.jar`, and puts it on
`implementation(files(...))`. `skitter-cli` (`main.sysl`) never sees the activity except as the name
`sh.sysl.skitter.SkitterActivity` (`ACTIVITY`, used by `adb shell am start`); it runs `./gradlew assembleDebug`,
`assembleRelease bundleRelease`, and on `clean` deletes `activity/target`. So the activity costs a second build
system (sbt), a second language, a dependency (`scala3-library_3:3.8.2`), and **sets `minSdk = 26`** (the stdlib
is what d8 will not desugar below 26; SDL's floor is 21 and sysl's triple is 24).

The code, with comments removed:

```scala
class SkitterActivity extends SDLActivity:
  @native def nativeSetSystemBars(left: Int, top: Int, right: Int, bottom: Int): Unit
  @native def nativePermissionAnswer(answer: Int): Unit

  override def onCreate(saved: Bundle): Unit =
    super.onCreate(saved)
    if !SDLActivity.mBrokenLibraries then
      val id = getResources.getIdentifier("skitter_keep_awake", "bool", getPackageName)
      val keepAwake = id != 0 && getResources.getBoolean(id)
      SDLActivity.nativeSetenv("SDL_VIDEO_ALLOW_SCREENSAVER", if keepAwake then "0" else "1")
    getWindow.getDecorView.setOnApplyWindowInsetsListener(new View.OnApplyWindowInsetsListener:   // anonymous class
      def onApplyWindowInsets(v: View, insets: WindowInsets): WindowInsets =
        if Build.VERSION.SDK_INT >= Build.VERSION_CODES.R then
          val bars = insets.getInsets(WindowInsets.Type.systemBars())
          nativeSetSystemBars(bars.left, bars.top, bars.right, bars.bottom)
          Log.i("skitter", s"system bars: ${List(...).zip(...).map(...).mkString(" ")}")   // a scala-library check
        insets)

  override def onUnhandledMessage(command: Int, param: AnyRef): Boolean =
    val n = param match { case i: Integer => i.intValue; case _ => 0 }
    command match
      case Vibrate => vibrate(n); true          // 0x8001
      case Tick => tick(); true                 // 0x8002
      case Share => share(); true               // 0x8003
      case CheckPermission => answerPermission(n); true   // 0x8004
      case _ => super.onUnhandledMessage(command, param)

  vibrator: Vibrator|null   // SDK >= S: getSystemService(VibratorManager.class).getDefaultVibrator else getSystemService(Vibrator.class); null unless hasVibrator
  vibrate(ms)   // if ms > 0: vibrator.vibrate(VibrationEffect.createOneShot(ms.toLong, DEFAULT_AMPLITUDE))
  tick()        // SDK >= Q: createPredefined(EFFECT_TICK) else createOneShot(10, DEFAULT_AMPLITUDE)
  share()       // text = SDLActivity.nativeGetHint("SKITTER_SHARE_TEXT"); if non-null: startActivity(Intent.createChooser(
                //   new Intent(ACTION_SEND).setType("text/plain").putExtra(EXTRA_TEXT, text), null))
  granted(p)    // p == "android.permission.BLUETOOTH_CONNECT" && SDK < S ? "android.permission.BLUETOOTH" : p;
                //   checkSelfPermission(asked) == PERMISSION_GRANTED
  answerPermission(token)   // p = nativeGetHint("SKITTER_PERMISSION"); nativePermissionAnswer(token*2 + (p != null && granted(p) ? 1 : 0))
```

**It is not tiny**, and the DSL below is sized to this, not to a stub: an anonymous inner class, a `switch`,
about twenty Android calls, SDK-level branches, one `null` test, one `instanceof`. The two `native` methods are
bound by name to `@export("Java_sh_sysl_skitter_SkitterActivity_...")` in `skitter/sh/sysl/skitter/insets.sysl` and
`phone.sysl`; renaming either side breaks at the first call, not at link time.

## 2. Why version 49, and the probe

The JVMS (§4.7.4) defines `StackMapTable` for class-file version 50 and later; §4.10.1 verifies by type checking
against it. Version 50 may fall back to the old inferring verifier (§4.10.2); **version 51 and later must have
frames**. A writer for 49 therefore emits no frames at all: constant pool, flags, fields, methods, `Code`
(`max_stack`, `max_locals`, bytes, an exception table only if wanted). `ldc` of a `Class` constant needs 49 (it is
a Java 5 feature), which is why the floor is 49 and not 48. Nothing newer is used: no `invokedynamic`, so string
concatenation is a `StringBuilder` chain.

Probe (`gen.py` in `/Users/ed/.claude/jobs/02a6d98d/tmp/skitter-note/`, a 40-line hand-assembler): a public class
`Probe49`, major 49, with `main` containing a branch (`aload_0 arraylength ifeq ... goto ...`, **no
`StackMapTable`**) and a `public static native nativeThing(IIII)V`.

```
java -cp cls Probe49          -> zero            (host JDK 27 loads and verifies it)
java -cp cls Probe49 x        -> nonzero
d8 --release --min-api 26 --lib android-36/android.jar --output out cls/Probe49.class   -> exit 0, no output
d8 --release --min-api 21 ...                                                             -> exit 0, no output
d8 --debug   --min-api 26 ...                                                             -> exit 0, no output
dexdump -d out/classes.dex    -> array-length / if-eqz / sget-object / invoke-virtual / goto; nativeThing is
                                 "access 0x0109 (PUBLIC STATIC NATIVE) code (none)"
```

Control: the same bytes with the version patched to 52: the JVM refuses (`VerifyError: Expecting a stackmap frame
at branch target 16`) and d8 warns `Expected stack map table for method with non-linear control flow` yet exits 0.
So **d8 does not demand frames at 49 and the JVM does not either**, and 52 is where frames start to matter. d8
printed no desugaring or min-API message at 21 or 26. With no Scala library in the APK the Scala-driven
`minSdk = 26` has no reason left; the floor becomes SDL's 21 or sysl's 24 (a separate decision, row 7).

## 3. The DSL

Indentation-structured, one file. Calls are resolved from `use` lines, each external member declared once, so the
generator needs no `android.jar` reader and no overload resolution (key: owner, name, arity).

```
file    := 'class' QName 'extends' QName { member }
member  := 'const' NAME '=' INT
         | 'use' ['static'] Owner '.' name '(' [Type {',' Type}] ')' Type      -- also: 'use' ['static'] Owner '.' FIELD ':' Type
         | 'native' name '(' [Type {',' Type}] ')'                              -- always void, always instance
         | ['override'] name '(' [Type id {',' Type id}] ')' [':' Type] block
         | 'inner' Name 'implements' QName { method }                           -- captures `outer`
stmt    := 'val' id '=' expr | 'var' id '=' expr | id '=' expr | expr
         | 'if' cond block ['else' block] | 'switch' expr { INT ':' block } 'else' block | 'return' [expr]
cond    := expr ('==' | '!=' | '<' | '>=' ...) expr | expr '==' 'null' | expr 'instanceof' Type | 'not' cond | cond ('and'|'or') cond
expr    := INT | STRING | 'null' | 'this' | 'outer' | id | NAME | Owner '.' FIELD | expr '.' name '(' args ')' | Owner '.' name '(' args ')'
         | 'new' Owner '(' args ')' | 'str' '(' args ')' | 'class' Owner | expr '+' expr | '(' Type ')' expr
```

`NAME` is a `const`; `str(a, b, ...)` is the `StringBuilder` chain; there is no overloading, no generics, no
expression `if`, no loops, no `try`. Because it is structured, the operand stack is empty at every statement
boundary, so `max_stack` is a running maximum over one expression and no join ever has to be reconciled.

Today's activity (the `scala-library` log line becomes a plain `Log.i("skitter", str("bars ", ...))`):

```
class sh.sysl.skitter.SkitterActivity extends org.libsdl.app.SDLActivity
const VIBRATE = 0x8001
const TICK = 0x8002
const SHARE = 0x8003
const CHECK_PERMISSION = 0x8004
use static org.libsdl.app.SDLActivity.mBrokenLibraries: boolean
use static org.libsdl.app.SDLActivity.nativeSetenv(String, String) void
use static org.libsdl.app.SDLActivity.nativeGetHint(String) String
use static android.os.Build$VERSION.SDK_INT: int
use android.content.Context.getResources() android.content.res.Resources
use android.content.Context.getPackageName() String
use android.content.res.Resources.getIdentifier(String, String, String) int
use android.content.res.Resources.getBoolean(int) boolean
use android.app.Activity.getWindow() android.view.Window
use android.view.Window.getDecorView() android.view.View
use android.view.View.setOnApplyWindowInsetsListener(android.view.View$OnApplyWindowInsetsListener) void
use android.view.WindowInsets.getInsets(int) android.graphics.Insets
use static android.view.WindowInsets$Type.systemBars() int
use android.graphics.Insets.left: int       -- and top, right, bottom
use android.util.Log.i(String, String) int  -- static; remaining Vibrator/Intent/PackageManager uses elided here

native nativeSetSystemBars(int, int, int, int)
native nativePermissionAnswer(int)

override onCreate(Bundle b)
  super.onCreate(b)
  if not SDLActivity.mBrokenLibraries
    val id = this.getResources().getIdentifier("skitter_keep_awake", "bool", this.getPackageName())
    var v = "1"
    if id != 0 and this.getResources().getBoolean(id)
      v = "0"
    SDLActivity.nativeSetenv("SDL_VIDEO_ALLOW_SCREENSAVER", v)
  this.getWindow().getDecorView().setOnApplyWindowInsetsListener(new Listener(this))

inner Listener implements android.view.View$OnApplyWindowInsetsListener
  onApplyWindowInsets(View v, WindowInsets w): WindowInsets
    if Build.SDK_INT >= 30
      val bars = w.getInsets(WindowInsets.systemBars())
      outer.nativeSetSystemBars(bars.left, bars.top, bars.right, bars.bottom)
    return w

override onUnhandledMessage(int command, Object param): boolean
  var n = 0
  if param instanceof Integer
    n = ((Integer) param).intValue()
  switch command
    VIBRATE: this.vibrate(n); return true
    TICK: this.tick(); return true
    SHARE: this.share(); return true
    CHECK_PERMISSION: this.answerPermission(n); return true
    else: return super.onUnhandledMessage(command, param)
-- vibrator(), vibrate(), tick(), share(), granted(), answerPermission() follow the same shapes.
```

Where the file lives and how a `native` signature is written:

- **Native signatures** use JVM type names the activity already uses (`int`, `boolean`, `String`, `Object`,
  `long`); the generator maps them to descriptors (`int` -> `I`, `String` -> `Ljava/lang/String;`, class names
  dotted -> slashed). The JNI symbol is derived from the class and method name by the generator
  (`Java_` + mangled `sh.sysl.skitter.SkitterActivity` + `_` + name), and **`skitter-cli` can print or check it**
  against the `@export` strings in the `skitter` package, which closes the silent-rename hole above.
- **Placement** (row 2 of the decisions): `activity/SkitterActivity.skitter` in the template (replaces the
  `.scala`, same place a reader looks), or embedded in the `skitter` binary.
  A block in `package.hocon` or `gradle.properties` is the worst fit: indentation-structured code in HOCON.

## 4. Bytecode per construct

All offsets are computed by the writer; every branch is forward, since the DSL has no loops. `depth` is the
operand-stack effect after the construct.

| Construct | Emits | depth |
|---|---|---|
| `int` literal | `iconst_n` (-1..5), `bipush`, `sipush`, else `ldc` of `CONSTANT_Integer` (`0x8001` is not a `short`) | +1 |
| string literal | `ldc` / `ldc_w` of `CONSTANT_String` (modified UTF-8 in the Utf8 entry) | +1 |
| `null`, `this`, `outer` | `aconst_null`; `aload_0`; `aload_0 getfield this$0` in an inner class | +1 |
| local read / write | `iload`/`aload n`, `istore`/`astore n` (`wide` past 255); params occupy `1..` | +1 / -1 |
| `Owner.FIELD` static / instance | `getstatic` / `getfield` (instance: receiver first) | +1 / 0 |
| instance call | receiver, args, `invokevirtual` (`invokeinterface` for an interface owner, with count byte) | -(args)(+1 if non-void) |
| static call | args, `invokestatic` | -(args)(+1 if non-void) |
| `super.m(...)` | `aload_0`, args, `invokespecial` on the superclass | as a call |
| `new C(args)` | `new`, `dup`, args, `invokespecial <init>` | +1 |
| `class C` | `ldc` of `CONSTANT_Class` (needs version 49) | +1 |
| `(T) e` / `e instanceof T` | `checkcast` / `instanceof` | 0 |
| `str(a, b)` | `new StringBuilder dup invokespecial <init>`, then `append` per part with the descriptor of the part's type, then `toString` | +1 |
| `a + b` (int) | `iadd`; a `long` operand is widened with `i2l` first, a `long` result takes 2 slots | -1 |
| `cond` as a branch | `if_icmpXX`, `ifXX`, `ifnull`/`ifnonnull`; `and`/`or` short-circuit by chaining jumps; `instanceof` then `ifeq` | 0 |
| `if c A else B` | `cond` jumping to Else; A; `goto End`; Else: B; End: | 0 at the boundary |
| `switch e` | `tableswitch` when the keys are dense (the four commands are), else `lookupswitch`; 0-3 pad bytes after the opcode | -1 |
| `return` / `return e` | `return` / `ireturn` / `areturn` | -(0 or 1) |
| `native m(...)` | method with `ACC_PUBLIC | ACC_NATIVE`, no `Code` attribute | n/a |
| `inner C implements I` | second class `Outer$1`: field `this$0`, constructor storing it then `invokespecial Object.<init>`, the interface method; a bridging method is needed only for generics, and the DSL has none | n/a |

`max_stack` is the running maximum of `depth` within a statement (always starting from 0); `max_locals` is
`1 + parameter slots + locals`. No exception table is needed by today's activity.

## 5. The split

- **`sysl-lang/classfile`** (module `sh.sysl.classfile`), public: the constant pool (dedup of Utf8, Class, String,
  NameAndType, Fieldref, Methodref, InterfaceMethodref, Integer, Long), modified UTF-8, descriptors, access flags,
  fields, methods, an **assembler** (labels, forward patching of branch offsets, `tableswitch`/`lookupswitch`
  padding, stack-depth and local-slot tracking), `ClassFile.write() -> Buf[u8]`, and a stored-entries jar writer.
  It knows nothing about Android or the DSL. A reader (for `javap`-style checks) is out of scope.
- **`sysl-lang/parsing`** (existing): the DSL's scanner, layout pass (indentation as structure), spans and
  diagnostics. Nothing DSL-specific is added there.
- **`skitter-cli`**: the DSL grammar, its checker (`use` resolution, types of locals, "unknown name" with a span),
  the lowering to the assembler (the table in section 4), the `skitter activity` command, and the JNI-symbol
  check against the `skitter` package.

## 6. Where it fits in the build

The Gradle `compileActivity` task stops running `sbt`; it runs `skitter activity --out app/build/skitter/activity.jar`
(or the CLI runs it first and Gradle reads the file). **sbt, `activity/build.sbt`, `scala3-library_3` and the
`SKITTER_CLASSPATH` plumbing all disappear**, `skitter clean` stops deleting `activity/target`, and `minSdk` may
drop. The DSL needs no `android.jar` (signatures are declared in `use` lines), so the step has no SDK dependency
and runs in milliseconds. AGP takes the jar through `implementation(files(...))` exactly as it takes the sbt jar
today; AGP's own d8 step turns it into dex (probe: section 2). The open point is how `./gradlew assembleDebug`
alone keeps working: the Gradle task can exec `skitter` (so the CLI must be on `PATH`, which brew gives) or the
template can carry a pre-generated jar (a binary in the repo, which the org avoids). Row 4.

## 7. Testing

1. **Round trip with `javap -v` as the oracle**: write each construct's class, run `javap -v -c`, and assert on the
   text (constant-pool entries, `stack=`/`locals=`, the instruction list). `javap` is in the JDK Gradle already
   needs; the test skips with a message where it is absent.
2. **Run on the host JDK**: generated classes with a `main` (a test-only DSL file with static helpers and no
   Android types) run under `java -Xverify:all -cp`, covering arithmetic, branches, `switch`, `str(...)`, `new`,
   casts. Version 49 loads on JDK 27 (probe).
3. **The activity itself under the verifier**: compile a few stub classes (`SDLActivity`, `Bundle`, ...) with the
   host `javac`, then `Class.forName` the generated activity with `-Xverify:all`. Skipped without `javac`.
4. **d8 as a second oracle**: `d8 --lib android.jar` over the generated jar must exit 0 with no `Warning`, and
   `dexdump` must show the two `native` methods with `code (none)`. Skipped without the SDK.
5. **Mutation pass**: break each opcode, each branch offset, and the descriptor of a `native`, and see a test go red.
6. **On device (the user)**: `skitter run` on the emulator or a phone; check the system-bars log line, keep-awake,
   a vibrate/tick, the share sheet, and a permission check. A wrong class fails at the first call, so one run of
   each path settles it.

## 8. Decisions for the user

| # | Question | Options | Recommendation | Why |
|---|---|---|---|---|
| 1 | A DSL, or the activity written in sysl against the assembler API? | DSL; a sysl `build_activity()` function; both | Assembler API first, DSL second | The activity is fixed Skitter machinery nobody edits, so a parser and `use` checker buy readability only; the DSL earns its keep once users add their own methods |
| 2 | Where does the source live? | template `activity/SkitterActivity.skitter`; embedded in `skitter`; in the `skitter` package | Template file | It sits where the `.scala` sits, a project can see it, and whether a package can ship a non-sysl asset was not checked |
| 3 | `use` lines vs. reading `android.jar` | declared once in the DSL; a class-file and zip reader resolving from the SDK | Declared | No reader, no inflate, no SDK needed to build; the cost is about 30 lines of `use` |
| 4 | How does `./gradlew assembleDebug` run the generator? | Gradle execs `skitter`; checked-in jar; `skitter build` only | Gradle execs `skitter` | Keeps the one-command promise; the CLI is already required |
| 5 | Jar writer | in `classfile` (stored zip + CRC-32); shell out to `jar` | In `classfile` | See gaps; about 60 lines; no JDK dependency in the generator |
| 6 | Does the `native` JNI symbol get checked by `skitter`? | derive and verify against the package; leave | Derive and verify | Turns today's silent `UnsatisfiedLinkError` into a build error |
| 7 | Lower `minSdk` once Scala is gone? | stay 26; 24; 21 | 24 | Matches sysl's triple; 21 would claim a floor the `.so` was not built against |
| 8 | Keep the Scala path as a fallback for projects that extend the activity? | remove; keep behind a flag | Remove | A second path is a second thing to keep in step; projects needing Java-side code can add a Java/Kotlin class beside it, which AGP already builds |
| 9 | Name and home of the package | `sysl-lang/classfile`, `sh.sysl.classfile`; inside `skitter-cli` | Own package | Generic machinery goes in its own package (org rule); a Dex or class reader may follow |

## 9. Gaps in sysl noticed

Checked in `~/dev/sysl-lang/sysl/library/sysl` on 2026-10-19:

- **Present**: `sysl.encoding.binary` has `put_u16_be`, `put_u32_be`, `put_u64_be` and the `get_*` pairs, each on a
  pre-sized `[]u8` returning `bool`. Growing a `Buf[u8]` and back-patching a length (the `Code` attribute length,
  a branch offset) works with these.
- **Gap, minor: no append-style big-endian writer on `Buf[u8]`** (`push_u16_be`, `push_u32_be`). The writer must
  grow the buffer and then `put_*` into a slice, which is the longhand a class-file, ELF or network writer repeats
  on every field. Reported under Rule 9; the classfile package would otherwise carry a private wrapper that every
  binary-format package copies.
- **Gap: no CRC-32** anywhere in `sysl` (`hash.sysl` has none) and **no zip/jar writer**. A stored-entries jar needs
  CRC-32. It belongs in std next to `sysl.hash`, not privately in `classfile`.
- **Gap, small: no modified UTF-8** (U+0000 as `C0 80`, supplementary characters as two three-byte surrogates).
  Ordinary strings are identical; only those two cases differ, so this is about 25 lines and belongs in
  `classfile` (JVM-specific) rather than std. Not a language gap.
- No gap in the language itself was found for the writer: byte buffers, integer widths, `match` on opcodes and
  arrays of labels are all expressible.
