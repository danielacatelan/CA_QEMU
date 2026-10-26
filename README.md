# CA_QEMU — QEMU com RISC-V (Iniciação Científica)

Registro, passo a passo, do que foi feito para montar o ambiente e **localizar onde as instruções RISC-V estão implementadas no código-fonte do QEMU**. O objetivo final do projeto é inserir **instruções aproximadas** no emulador.

## Sumário

1. [Visão geral do ambiente](#1-visão-geral-do-ambiente)
2. [Instalar o QEMU no Windows (binário)](#2-instalar-o-qemu-no-windows-binário)
3. [Instalar o WSL2 com Ubuntu](#3-instalar-o-wsl2-com-ubuntu)
4. [Baixar o código-fonte do QEMU](#4-baixar-o-código-fonte-do-qemu)
5. [Onde ficam as instruções RISC-V](#5-onde-ficam-as-instruções-risc-v)
6. [Abrir e editar os arquivos](#6-abrir-e-editar-os-arquivos)
7. [Problemas encontrados](#7-problemas-encontrados)
8. [Próximos passos](#8-próximos-passos)

---

## 1. Visão geral do ambiente

| Item | Valor |
|---|---|
| Sistema operacional | Windows 11 |
| QEMU no Windows | 11.1.0 (`v11.1.0-12130-ge470268ff4`), binário pré-compilado |
| Linux | WSL2, distribuição Ubuntu 24.04 |
| Código-fonte | Repositório oficial `github.com/qemu/qemu` |

São dois ambientes com funções diferentes:

- **QEMU instalado no Windows:** serve para *rodar* sistemas RISC-V e ter um QEMU "original" para comparação. Não contém o código-fonte.
- **WSL2 (Ubuntu):** é onde o código-fonte fica, para ser lido, editado e, mais adiante, compilado.

## 2. Instalar o QEMU no Windows (binário)

Seguido o tutorial em vídeo: https://www.youtube.com/watch?v=CstEdMjx4PY

O instalador coloca os programas em `C:\Program Files\qemu` (no Windows em português aparece como `C:\Arquivos de Programas\qemu`). Entre eles estão `qemu-system-riscv64.exe`, `qemu-system-riscv32.exe` e `qemu-img.exe`.

Para conferir a versão, no PowerShell:

```powershell
& "C:\Program Files\qemu\qemu-system-riscv64.exe" --version
```

Saída obtida:

```
QEMU emulator version 11.1.0 (v11.1.0-12130-ge470268ff4)
Copyright (c) 2003-2026 Fabrice Bellard and the QEMU Project developers
```

> O QEMU do Windows **não** traz o modo usuário (`qemu-riscv64`), que existe apenas no Linux. Ele também não traz a pasta `target/riscv/`, por isso o código-fonte é baixado à parte (seção 4).

## 3. Instalar o WSL2 com Ubuntu

Em um PowerShell **como administrador**:

```powershell
wsl --install -d Ubuntu-24.04
```

Reinicie o computador se for pedido. Na primeira abertura, o Ubuntu pede para criar um **usuário** e uma **senha**. A senha não aparece na tela enquanto é digitada; é assim mesmo. Ela será pedida pelo `sudo`.

Para confirmar que a distribuição usa a versão 2:

```powershell
wsl -l -v
```

Saída obtida (a distribuição usada neste projeto é a `Ubuntu-24.04`; o `docker-desktop` é interno do Docker e não deve ser mexido):

```
  NAME              STATE      VERSION
* Ubuntu            Stopped    2
  docker-desktop    Stopped    2
  Ubuntu-24.04      Stopped    2
```

Para entrar no Ubuntu:

```powershell
wsl -d Ubuntu-24.04
```

O prompt muda para algo como `acer@NOME-DO-PC:~$`. **Todos os comandos das próximas seções são executados dentro desse prompt**, e não no PowerShell.

## 4. Baixar o código-fonte do QEMU

Dentro do Ubuntu:

```bash
sudo apt update && sudo apt install -y git
cd ~
git clone --depth 1 https://github.com/qemu/qemu.git
cd qemu
```

- `--depth 1` baixa apenas a versão mais recente, sem o histórico (muito mais rápido).
- O clone deve ficar sempre dentro do Linux (`~`), **nunca** em `/mnt/c/...`, que é muito lento.
- Este `git clone` apenas baixa o código; nada é registrado em conta do GitHub.

Para trabalhar com uma versão específica (por exemplo, a mais próxima do binário do Windows), pode-se usar uma tag:

```bash
git clone --depth 1 --branch v11.1.0 https://github.com/qemu/qemu.git
```

## 5. Onde ficam as instruções RISC-V

Tudo está na pasta **`target/riscv/`**:

```bash
cd ~/qemu
ls target/riscv/
ls target/riscv/insn_trans/
```

| Arquivo | Função |
|---|---|
| `insn32.decode` | Define cada instrução de 32 bits (nome, padrão de bits, formato) |
| `insn16.decode` | Instruções comprimidas (extensão C) |
| `insn_trans/trans_rvi.c.inc` | Instruções inteiras básicas (soma, lógica, loads, stores) |
| `insn_trans/trans_rvm.c.inc` | Multiplicação e divisão |
| `insn_trans/trans_rvf.c.inc`, `trans_rvd.c.inc` | Ponto flutuante (float e double) |
| `insn_trans/trans_rvv.c.inc` | Vetorial |
| `translate.c` | Laço de tradução e lista de decodificadores |
| `helper.h` | Declaração dos helpers (funções C chamadas pelo código traduzido) |
| `op_helper.c`, `fpu_helper.c`, `vector_helper.c` | Implementação dos helpers |
| `cpu.c`, `cpu_cfg.h` | Registro das extensões e opções da CPU |
| `meson.build` | Configuração de build |

> Os nomes foram listados com base no conhecimento geral do projeto e podem variar entre versões. A fonte confiável é o `ls` acima, na versão baixada.

### Seguindo uma instrução do começo ao fim

Exemplo com `mul` (inteiro) e `fadd.s` (ponto flutuante):

```bash
# 1) Definição (padrão de bits)
grep -n "^mul\b" target/riscv/insn32.decode
grep -n "^fadd_s" target/riscv/insn32.decode

# 2) Função de tradução
grep -rn "trans_mul\b" target/riscv/
grep -rn "trans_fadd_s" target/riscv/

# 3) Helper chamado (se houver)
grep -rn "helper_fadd_s" target/riscv/
```

Cadeia esperada: `insn32.decode` → função `trans_<nome>` em `insn_trans/*.c.inc` → operação TCG direta **ou** helper em `*_helper.c`.

Também é possível navegar o código pelo navegador, sem instalar nada (mostra o `master`, não a versão exata): https://github.com/qemu/qemu/tree/master/target/riscv

## 6. Abrir e editar os arquivos

Comandos como `cat` e `less` apenas **mostram** o arquivo; não permitem edição. Para editar, dentro do Ubuntu:

**VS Code (recomendado).** Requer o VS Code no Windows com a extensão **WSL** (Microsoft):

```bash
cd ~/qemu
code .
```

**nano (no terminal):**

```bash
cd ~/qemu/target/riscv
nano insn32.decode
```

Atalhos do nano: `Ctrl+O` e Enter salvam, `Ctrl+X` sai, `Ctrl+W` procura.

Regra adotada: **não alterar os arquivos originais**; as instruções aproximadas serão criadas em arquivos novos. Para desfazer uma edição acidental:

```bash
git checkout -- target/riscv/NOME_DO_ARQUIVO
```

## 7. Problemas encontrados

| Problema | Causa | Solução |
|---|---|---|
| `git checkout e470268ff4` → `pathspec did not match` | O commit do binário do Windows não existe no repositório oficial | Usar uma tag próxima (`v11.1.0`) ou o `master`; para explorar as instruções, a diferença de versão não importa |
| `cd ~/qemu` → `Não é possível localizar o caminho 'C:\Users\...\qemu'` | O comando foi executado no PowerShell do Windows, e não dentro do Ubuntu | Entrar antes no Ubuntu com `wsl -d Ubuntu-24.04` |
| Não consegue editar o arquivo | Foi aberto com `less`/`cat`, que são somente leitura | Usar `code .` ou `nano` |
| Erro de permissão ao salvar | Arquivos pertencem ao `root` ou estão em `/mnt/c` | `sudo chown -R $USER:$USER ~/qemu` e manter o código em `~` |

## 8. Próximos passos

- [ ] Escolher a operação a aproximar (soma, multiplicação ou ponto flutuante)
- [ ] Definir com o orientador a técnica de aproximação
- [ ] Compilar o QEMU para RISC-V no WSL
- [ ] Criar a primeira instrução aproximada no espaço *custom* do RISC-V (opcodes `custom-0` = `0x0B`, `custom-1` = `0x2B`, `custom-2` = `0x5B`, `custom-3` = `0x7B`), usando as extensões já existentes no código como modelo
- [ ] Escrever programas de teste em C e comparar com o QEMU original do Windows
- [ ] Guardar neste repositório apenas documentação, testes e *patches* (o código-fonte do QEMU não deve ser copiado para cá)
