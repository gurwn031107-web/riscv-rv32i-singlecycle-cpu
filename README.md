# RV32I Single Cycle CPU

SystemVerilog로 설계한 RISC-V RV32I 싱글사이클 CPU입니다.
한 클럭에 명령어 하나를 가져와서 해석하고, 실행하고, 결과를 레지스터나 메모리에 쓰는 구조로 R / I / S / L / B / U / J 타입 명령어를 지원합니다.
이 CPU를 멀티사이클로 바꾸고 APB 주변장치를 붙인 다음 단계는 [RV32I Multi-Cycle CPU + UVM](https://github.com/gurwn031107-web/proj_rv32i_multi_cycle_cpu) 팀 프로젝트로 이어졌습니다.

- 언어 / 도구: SystemVerilog, Vivado (Behavioral Simulation)

## 구조

```
           ┌────────────── Control Unit ───────────────┐
           │ opcode, funct3, funct7 → ALU 제어,         │
           │ ALU 입력 선택, WB 선택, 메모리 쓰기, itype  │
           └───────────────────────────────────────────┘
PC ──> Instruction ROM ──> Register File ──> ALU ──> Data Memory
 ▲                            ▲                │          │
 │        Immediate Extend ───┘ (ALU 입력 MUX) │          │
 │                                             ▼          ▼
 └── PC + 4 / PC + imm / rs1 + imm      WB MUX (ALU, 메모리, imm, PC + imm, PC + 4)
```

| 모듈 | 하는 일 |
|---|---|
| `control_unit` | opcode로 명령 타입을 구분하고 ALU 동작, ALU 두 번째 입력(rs2 / imm), 쓰기 데이터 선택, 메모리 쓰기 신호를 만듦 |
| `datapath` | PC, 레지스터 파일(32 x 32bit, x0 = 0 고정), ALU, 분기 비교, 즉시값 확장, WB MUX |
| `imm_extend` | I / S / B / U / J 타입별로 흩어진 즉시값 비트를 모아 부호 확장 |
| `data_memory` | 바이트 / 하프워드 / 워드 단위 쓰기와 부호 / 무부호 읽기 (SB, SH, SW, LB, LH, LW, LBU, LHU) |
| `instruction_rom` | 테스트 프로그램 (LUI, AUIPC, JAL 호출 / 복귀, SW, 분기) |

## 지원 명령어

| 타입 | 명령어 |
|---|---|
| R | ADD, SUB, SLL, SLT, SLTU, XOR, SRL, SRA, OR, AND |
| I (연산) | ADDI, SLTI, SLTIU, XORI, ORI, ANDI, SLLI, SRLI, SRAI |
| L (로드) | LB, LH, LW, LBU, LHU |
| S (저장) | SB, SH, SW |
| B (분기) | BEQ, BNE, BLT, BGE, BLTU, BGEU |
| U | LUI, AUIPC |
| J | JAL, JALR |

- WB MUX는 다섯 가지 값 중 하나를 레지스터에 씁니다: ALU 결과, 메모리 읽기 값, 즉시값(LUI), PC + imm(AUIPC), PC + 4(JAL / JALR의 복귀 주소)
- JALR은 rs1 + imm의 최하위 비트를 0으로 만든 주소로 점프합니다.

## 테스트벤치

| 파일 | 확인 내용 |
|---|---|
| `tb/tb_rom.sv` | ROM 프로그램을 CPU와 데이터 메모리로 실행해 U 타입, 호출 / 복귀, 분기, 저장 결과 확인 |
| `tb/tb_u_type.sv` | LUI, AUIPC 결과값, x0에 쓰기 무시, ADD / ADDI / LW 회귀, BEQ / BNE 분기 여부 |
| `tb/tb_jump.sv` | JAL / JALR 호출과 복귀, 음수 오프셋, rd와 rs1이 같은 경우, J 타입 즉시값 경계, 잘못된 funct3 무시 |

각 테스트벤치는 기대값과 다르면 바로 멈추고, 모두 맞으면 `PASS` 메시지를 출력합니다.

## 실행

Vivado에서 새 프로젝트를 만들고 `rtl/` 아래 파일(패키지 `rtl/pkg/rv32i_pkg.sv` 포함)과 `tb/` 파일 하나를 추가한 뒤 Behavioral Simulation을 실행합니다.
