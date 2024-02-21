# Simulation

Chipyard 支持两类n仿真：

1. 使用商业或开源 (Verilator) RTL 仿真器进行软件 RTL 仿真。
2. 使用 FireSim 进行 FPGA 加速的全系统仿真。

Chipyard 设计的软件 RTL 仿真器运行速度为 O(1 KHz)，但编译速度很快并提供完整波形。相反，FPGA 加速模拟器的运行速度为 O(100 MHz)，适合启动操作系统和运行完整的工作负载，但编译时间长达数小时且调试可视性较差。