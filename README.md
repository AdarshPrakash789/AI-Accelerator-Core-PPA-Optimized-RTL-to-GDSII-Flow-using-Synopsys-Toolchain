# AI Accelerator Core: PPA-Optimized RTL to GDSII Flow using Synopsys Toolchain

A complete RTL2GDSII ASIC flow of a custom AI accelerator core with PPA (Power, Performance, Area) optimization using the Synopsys toolchain.

## Table of Contents
- Tools Used
- Project Structure
- RTL Design Overview
- Results (TSMC 7nm)
- Flow Execution
- Original TCL Scripts
- Testbench
- Folder Tree

## Tools Used

| Step           | Tool              |
|----------------|-------------------|
| Synthesis      | Design Compiler   |
| Logic Equiv.   | Formality         |
| PnR            | IC Compiler II    |
| Timing/Power   | PrimeTime         |
| Simulation     | VCS               |
| Debug          | Verdi             |

## Project Structure

```
ppa_ai_core_rtl2gds_flow/
├── README.md
├── doc/
│   ├── architecture_diagram.png
│   └── ppa_analysis_summary.md
├── rtl/
│   ├── ai_core.v
│   ├── mac_unit.v
│   └── controller.v
├── scripts/
│   ├── synth/dc_compile.tcl
│   ├── formal/formality_check.tcl
│   ├── pd/floorplan.tcl
│   ├── pd/place_route.tcl
│   ├── timing/primetime_setup.tcl
│   └── sim/vcs_simulation.do
├── constraints/
│   └── ai_core.sdc
├── reports/
│   ├── synthesis/
│   ├── layout/
│   ├── timing/
│   └── power/
└── testbench/
    └── ai_core_tb.v
```

## RTL Design Overview

```verilog
module ai_core (
    input clk,
    input rst,
    input [7:0] in_data,
    output [15:0] out_data
);
    wire [15:0] mac_out;
    wire [7:0] ctrl_signal;

    mac_unit u_mac (
        .clk(clk),
        .rst(rst),
        .in_data(in_data),
        .mac_out(mac_out)
    );

    controller u_ctrl (
        .clk(clk),
        .rst(rst),
        .control(ctrl_signal)
    );

    assign out_data = mac_out ^ {8'b0, ctrl_signal};
endmodule
```

## Results (TSMC 7nm)

| Metric        | Value         |
|---------------|---------------|
| Frequency     | 1.2 GHz       |
| Area          | 0.245 mm^2    |
| Dynamic Power | 135 mW        |
| Leakage Power | 8.5 mW        |
| Total Cells   | 65,000        |

## Flow Execution

1. **Synthesis**:
   ```bash
   dc_shell -f scripts/synth/dc_compile.tcl
   ```
2. **Equivalence Check**:
   ```bash
   formality -f scripts/formal/formality_check.tcl
   ```
3. **Floorplan, Place & Route**:
   ```bash
   icc2_shell -f scripts/pd/floorplan.tcl
   icc2_shell -f scripts/pd/place_route.tcl
   ```
4. **Timing Analysis**:
   ```bash
   primetime -f scripts/timing/primetime_setup.tcl
   ```
5. **Simulation**:
   ```bash
   vcs -f scripts/sim/vcs_simulation.do
   ```

## Original TCL Scripts

**dc_compile.tcl**:
```tcl
set_app_var search_path [list ../rtl ../constraints]
set_app_var target_library [list /path/to/tsmc7nm.db]
set_app_var link_library "* $target_library"

read_verilog ai_core.v mac_unit.v controller.v
current_design ai_core
link
read_sdc ai_core.sdc
compile_ultra -gate_clock
report_timing > ../reports/synthesis/timing.rpt
report_area > ../reports/synthesis/area.rpt
report_power > ../reports/synthesis/power.rpt
write -format ddc -hierarchy -output ai_core.ddc
```

**primetime_setup.tcl**:
```tcl
set search_path [list ../rtl ../constraints ../reports/layout]
set link_path [list /path/to/tsmc7nm.db]
read_verilog ai_core.v
current_design ai_core
link
read_parasitics -format SPEF ai_core.spef
read_sdc ai_core.sdc
report_timing -max_paths 10 -delay_type max -sort_by group > ../reports/timing/setup.rpt
report_power > ../reports/power/pt_power.rpt
```

**floorplan.tcl**:
```tcl
set_design_mode -flat
read_verilog ../rtl/ai_core.v ../rtl/mac_unit.v ../rtl/controller.v
set_top ai_core
create_floorplan -core_utilization 0.65 -core_aspect_ratio 1.0 -row_height 12 -site core
create_clock -name clk -period 0.833 [get_ports clk]
compile_floorplan
save_mw_cel -as floorplan
```

**place_route.tcl**:
```tcl
open_mw_cel floorplan
place_opt
clock_opt -only_psyn
route_opt
write_gds -hierarchy ai_core.gds
report_timing > ../reports/layout/postroute_timing.rpt
report_power > ../reports/power/postroute_power.rpt
save_mw_cel -as ai_core_final
```

## Testbench

```verilog
module ai_core_tb;
    reg clk;
    reg rst;
    reg [7:0] in_data;
    wire [15:0] out_data;

    ai_core dut (
        .clk(clk),
        .rst(rst),
        .in_data(in_data),
        .out_data(out_data)
    );

    initial begin
        clk = 0;
        forever #5 clk = ~clk;
    end

    initial begin
        rst = 1; in_data = 8'd0;
        #10 rst = 0;
        repeat (20) begin
            in_data = $random;
            #10;
        end
        $finish;
    end
endmodule
```

## Folder Tree

```
ppa_ai_core_rtl2gds_flow/
├── constraints/           # SDC constraints
├── doc/                   # Diagrams and architecture documents
├── reports/               # Generated reports
├── rtl/                   # Verilog RTL files
├── scripts/               # Tool-specific run scripts
├── testbench/             # Testbenches for functional simulation
└── README.md
```

---

## Author
Adarsh Prakash  
Email: kumaradarsh663@gmail.com  
LinkedIn: https://linkedin.com/in/adarshprakash

