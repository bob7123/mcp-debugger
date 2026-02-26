# Hull Moving Average (HMA) Indicator for NinjaTrader 8

## What Is the Hull Moving Average?

The Hull Moving Average (HMA), developed by Alan Hull in 2005, is a technical
indicator that dramatically reduces the lag inherent in traditional moving
averages while maintaining curve smoothness. It achieves this by combining
Weighted Moving Averages (WMA) at different time scales.

### Formula

Given a period **n**:

1. **WMA half** = WMA(Close, n/2)
2. **WMA full** = WMA(Close, n)
3. **Delta** = 2 &times; WMA_half &minus; WMA_full
4. **HMA** = WMA(Delta series, &radic;n)

The "2 &times; half &minus; full" trick amplifies recent price action, and the
final WMA over &radic;n bars smooths the result without reintroducing lag.

## NinjaTrader 8 Compilation Architecture

NinjaTrader 8 has its own built-in C# compiler. There is **no `.csproj` file**.
You place `.cs` files in the appropriate subfolder under
`Documents\NinjaTrader 8\bin\Custom\` and NinjaTrader compiles **all**
NinjaScript files into a single `Custom.dll` assembly. Key points:

- Indicators go in `bin\Custom\Indicators\`
- Strategies go in `bin\Custom\Strategies\`
- NinjaTrader compiles ALL `.cs` files, not just the one you're editing
- Compile errors from other files will appear alongside yours
- No external build tool or project file is needed

## Creating the Indicator

### Class Structure

A NinjaTrader 8 indicator inherits from `Indicator` and implements:

- **`OnStateChange()`** — Called during lifecycle transitions (SetDefaults,
  Configure, DataLoaded). Set the indicator name, default period, and plot
  appearance in `SetDefaults`. Pre-compute derived values (halfPeriod,
  sqrtPeriod) in `DataLoaded`.
- **`OnBarUpdate()`** — Called on each bar. This is where the four-step HMA
  algorithm runs.

### Key Implementation Details

- **Namespace**: `NinjaTrader.NinjaScript.Indicators`
- **Class name**: `HullMA`
- **Period property**: Expose as `[NinjaScriptProperty]` with `[Range(1, int.MaxValue)]`.
- **Plot**: Add a single overlay plot (`IsOverlay = true`) in SetDefaults.
- **Ring buffer**: Use a `double[]` of length `sqrt(period)` to hold the delta
  series. Shift right on each bar, insert new delta at index 0.
- **WMA helper**: `weight = len - i` where `i = 0` is the newest bar. Sum
  `Input[i] * weight`, divide by total weight.
- **Warmup**: Skip the HMA calculation until `CurrentBar >= Period`. Output
  `Input[0]` (raw price) during warmup.

### Installing

Place `HullMA.cs` in:
```
%USERPROFILE%\Documents\NinjaTrader 8\bin\Custom\Indicators\HullMA.cs
```

Then compile inside NinjaTrader:
1. Open NinjaTrader 8.
2. Go to **Tools &rarr; Edit NinjaScript &rarr; Indicators**.
3. Press **F5** to compile (or right-click &rarr; Compile).
4. Add to a chart: right-click chart &rarr; **Indicators** &rarr; **HullMA** &rarr;
   **Add** &rarr; set **Period** &rarr; **OK**.

## Debugging with mcp-debugger

Because NinjaTrader compiles indicators into `Custom.dll` inside its own
runtime, you cannot attach a standalone debugger to the indicator directly.
The recommended approach is to create a standalone .NET 8 console app
(`HullMAConsole.cs`) that extracts the same algorithm into plain C# with
sample price data. This console app can then be debugged with mcp-debugger.

[mcp-debugger](https://github.com/debugmcp/mcp-debugger) is an MCP server
that gives AI agents (like Claude) step-through debugging via the Debug
Adapter Protocol. It supports Python, JavaScript, Rust, Go, and **.NET/C#**.

### Division of Labor

This example uses a **two-agent workflow**:

1. **NinjaTrader agent** — Starts in `%USERPROFILE%\Documents\NinjaTrader 8`.
   Has access to `nt8-docs/` (scraped NinjaTrader documentation) and the full
   NT8 directory tree.
   - Places `HullMA.cs` in `bin\Custom\Indicators\`
   - Creates `HullMAConsole.cs` (standalone debuggable app) in the `claude\`
     working directory
   - Calls the mcp-debugger MCP tools to set breakpoints, step through, and
     inspect variables
   - Stores notes, metadata, and working files in `claude\`
   - If a bug is found in the debugger, writes the issue to
     `claude\mcp-debugger-issues.md`

2. **MCP debugger agent** — Starts in the mcp-debugger repo. Maintains the
   debugger server and its .NET adapter. Does NOT create indicator source
   files. Checks for `bug.md` from the NinjaTrader agent and fixes debugger
   bugs. Checks `%USERPROFILE%\Documents\NinjaTrader 8\claude\mcp-debugger-issues.md`
   for reported problems.

### Suggested Breakpoint Lines

When creating `HullMAConsole.cs`, mark these as good breakpoint candidates:

- The line calling `CalculateWMA` for the half-period (inspect `wmaHalf`)
- The delta calculation: `2 * wmaHalf - wmaFull` (inspect all three values)
- The final HMA from `CalculateWMAFromBuffer` (inspect `hma`)

### What to Verify

- **Variable values**: At the delta line, confirm `delta == 2*wmaHalf - wmaFull`.
- **Buffer state**: After `ShiftBuffer`, inspect the delta ring buffer.
- **Edge cases**: First bars after warmup will have a mostly-zero delta buffer,
  producing dampened HMA values. This is expected.

## Algorithm Notes

- The **warmup period** is `period - 1` bars. During warmup, output the raw
  price instead of an HMA value.
- Even after warmup, the delta ring buffer takes an additional `sqrt(period)`
  bars to fill completely. Early HMA values will be dampened.
- Weights in WMA go from highest (newest bar) to lowest (oldest bar):
  `weight = len - i` where `i = 0` is the most recent bar.
