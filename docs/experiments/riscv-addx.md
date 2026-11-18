# Experimento local: instrução RISC-V `addx`

> **Status:** protótipo local, com validação funcional pendente. Este documento
> registra o experimento no repositório CA_QEMU; não é uma contribuição ao QEMU
> upstream. Consulte a [política de proveniência do QEMU](https://gitlab.com/qemu-project/qemu/-/blob/master/docs/devel/code-provenance.rst)
> antes de preparar qualquer contribuição ao projeto.

## Objetivo e estado atual

O experimento adiciona ao clone local do QEMU uma instrução de 32 bits chamada
`addx`, usando o opcode customizado `custom-1` (`0x2B`). Na implementação atual,
`addx` calcula XOR entre os operandos. Isso é o comportamento observado no
código e esperado pelo teste; não significa que a operação pretendida ou sua
aproximação já tenham sido especificadas.

## Alterações no clone do QEMU

- `target/riscv/insn32.decode`: adiciona o padrão R-type para opcode `0x2B`,
  com `funct7 = 0` e `funct3 = 0`.
- `target/riscv/tcg/insn_trans/trans_rvi.c.inc`: adiciona `trans_addx`, usando
  `gen_logic` com `tcg_gen_xor_tl`.
- `tests/tcg/riscv64/test-addx.c`: emite a instrução por meio de `.insn` e
  compara os resultados de `add` e `addx`.
- `tests/tcg/riscv64/meson.build`: inclui `test-addx.c` na lista de testes TCG
  de `riscv64`.

O código-fonte do QEMU não foi copiado para este repositório; os caminhos acima
referem-se ao clone de trabalho.

## Casos do teste

O teste usa dois pares de operandos:

| Operandos | `add` esperado | `addx` esperado atualmente |
|---|---:|---:|
| `5, 10` | `15` | `15` |
| `5, 3` | `8` | `6` |

O segundo caso diferencia XOR de soma: `5 ^ 3` resulta em `6`, enquanto
`5 + 3` resulta em `8`.

## Validação pendente

Foi tentada a execução focada com:

```console
build-riscv/run meson test -C build-riscv test-addx --print-errorlogs
```

O build disponível retornou `*:test-addx test name does not match any test`.
Assim, a execução funcional ainda não foi confirmada; a presença do arquivo de
teste e sua inclusão no Meson não comprovam que o teste passa.

## Próximas decisões

1. Confirmar com a orientação do projeto o comportamento pretendido para
   `addx` e se XOR é mesmo a operação desejada.
2. Atualizar ou configurar o build para que o teste seja descoberto pelo Meson.
3. Executar o teste e registrar o resultado antes de considerar o protótipo
   validado.
