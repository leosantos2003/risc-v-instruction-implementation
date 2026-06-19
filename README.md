# Implementação de Instruções na Arquitetura RISC-V

**INF01113 - Organização de Computadores - UFRGS - 2026/1**

Alunos: Henrique Anderson Knorst, Henrique de Lima Bortolomiol, Leonardo Rocha dos Santos e Leônidas Lewy de Paula dos Santos

## 1. Introdução

Este relatório descreve as modificações realizadas no bloco operativo e nos bits de controle dos processadores para suportar novas instruções. Foram implementadas as instruções pertinentes ao Grupo 3: JALR, BGE, LB e DIV. As alterações foram replicadas e adaptadas para as três organizações do RISC-V estudadas: Monociclo, Multiciclo e Pipeline.  

## 2. Implementação das Instruções

### 2.1. Instrução JALR

- **Monociclo:** O endereço de salto (jump) é calculado e, caso a condição seja satisfeita, o valor de PC+4 é armazenado no banco de registradores, enquanto o endereço de desvio é carregado no PC. Para permitir este fluxo, foram adicionados dois multiplexadores (MUX): o primeiro encaminha o valor de OldPC para o banco de registradores, e o segundo carrega o endereço de desvio no PC. Adicionou-se também o sinal de controle JumpWrite, o qual teve seus bits configurados no endereço 0x67. A inversão lógica (bit flip) foi realizada utilizando uma porta AND combinada com uma constante.  

- **Multiciclo:** De maneira análoga ao monociclo, incluíram-se dois MUXes: um para enviar o OldPC ao banco de registradores e outro para enviar o endereço de desvio ao PC. Foi adicionado um sinal de controle de salto e criado um novo estado na máquina de controle, alocado na posição 9, contendo o valor 0x8a82. A execução do salto ocorre no estágio 3, estabelecendo a seguinte transição de estados: 0 -> 1 -> 2 -> 9.
    - **Diagrama de Estados (0 -> 1 -> 2 -> 9):** O fluxo inicia-se com a busca e a decodificação habituais. No Estado 2, realiza-se o cálculo do endereço de destino da ramificação. Na sequência, a execução converge para o Estado 9 (um estado customizado que assume o valor de controle 0x8a82) , ciclo no qual o endereço de salto é carregado efetivamente no PC e o valor do OldPC é armazenado no banco de registradores.

- **Pipeline:** O cálculo matemático do endereço é realizado no terceiro estágio. O sinal de JumpWrite é gerado durante a busca (fetch) da instrução, e no terceiro estágio a instrução é efetivamente executada, reaproveitando o endereço de escrita por meio do MUX inferior. Foram inseridos dois MUXes: um posicionado no terceiro estágio para receber o OldPC e armazená-lo no banco de registradores, e outro conectado ao PC para receber o novo endereço calculado. Além disso, implementou-se uma porta lógica OR para antecipar a geração do sinal de escrita nos registradores diretamente no terceiro estágio, em vez de aguardar o estágio de write-back.  

## 2.2. Instrução BGE

- **Modificações Gerais:** A implementação reaproveitou o sinal de branch pré-existente. A validação da instrução BGE é realizada por meio da avaliação do campo funct3, que deve ser igual a 5. Quando a condição do BGE é satisfeita (resultado positivo), o processador utiliza a mesma lógica matemática já implementada para a instrução BEQ. Para suportar as operações do BGE, a Unidade Lógica e Aritmética (ULA) foi modificada para expor as flags de Sinal (Sign) e Overflow.
    - **Diagrama de Estados (0 -> 1 -> 3):** Esta instrução reaproveita a estrutura de desvio condicional do processador (semelhante ao BEQ). No Estado 3, a lógica de controle valida se o campo funct3 corresponde ao valor 5 e analisa as flags de Sinal (Sign) e Overflow disponibilizadas pela ULA para determinar se a condição de salto foi atendida

- **Pipeline:** Considerando que no pipeline o cálculo é efetuado no terceiro estágio e o desvio ocorre apenas no quarto estágio, foram adicionados dois registradores para propagar o valor de OldPC até o terceiro estágio. Complementarmente, inseriu-se um flip-flop com a finalidade de reter a flag de condição do BGE até o quarto estágio, momento no qual o salto é concretizado.  

### 2.3. Instrução LB

- **Monociclo e Pipeline:** O caminho de dados da instrução LW foi reaproveitado, uma vez que a operação na ULA para o cálculo do endereço (soma do registrador base com o imediato) é idêntica para ambas as instruções. Desenvolveu-se um circuito lógico dedicado que avalia o opcode de load simultaneamente ao funct3 igual a 0, gerando a flag de controle isLB. Esta flag controla um novo MUX posicionado imediatamente após a saída da memória de dados: se a instrução for LW, a palavra completa de 32 bits é repassada; se for LB, o circuito extrai o byte correspondente utilizando os bits [1:0] do endereço e o encaminha para um extensor de sinal. No Pipeline, a flag isLB é simplesmente propagada pelos registradores de barreira ID/EX e EX/MEM, garantindo que atinja o MUX no estágio correto de memória.  

- **Multiciclo:** O LB utiliza a mesma máquina de estados do LW, executando os passos nos mesmos 5 ciclos. Aplicou-se a mesma arquitetura do MUX com extensor de sinal na saída do Registrador de Dados da Memória (MDR). Como a ULA é um recurso compartilhado nesta arquitetura, o valor do endereço calculado acabava sendo sobrescrito em ciclos subsequentes. Para solucionar este problema, foi adicionado um pequeno registrador de 2 bits, habilitado pelo sinal MemRead. Este componente armazena os bits [1:0] do endereço calculado pela ULA, assegurando que, no estado de write-back, o MUX tenha a referência precisa de qual byte da palavra deve ser extraído e armazenado no banco de registradores.
    - **Diagrama de Estados (0 -> 1 -> 2 -> 4 -> 5):** Segue estritamente a mesma sequência de 5 ciclos da instrução LW. No Estado 2, calcula-se o endereço alvo. No Estado 4 (leitura da memória), ativa-se o sinal MemRead e armazena-se os bits de endereço [1:0] em um registrador secundário de 2 bits. Por fim, no Estado 5 (write-back), esse valor preservado orienta o multiplexador a extrair apenas o byte selecionado, realizando a extensão de sinal antes de gravá-lo no banco de registradores.

### 2.4. Instrução DIV

- **Modificação Geral (Monociclo, Multiciclo e Pipeline):** Foi desenvolvido um subcircuito dedicado exclusivamente à operação de divisão no interior da ULA. A divisão básica utiliza o bloco lógico padrão do Logisim; entretanto, para adequar o resultado ao padrão da arquitetura RISC-V, adicionou-se uma lógica de tratamento de exceções. Quando ocorre overflow, o circuito retorna a própria entrada, e em cenários de divisão por zero, o circuito retorna o valor -1, conforme estipulado pelo manual da arquitetura. Como a estrutura da ULA é mantida consistente entre os três processadores, essas alterações foram replicadas de forma exata nas arquiteturas Monociclo, Multiciclo e Pipeline.
    - **Diagrama de Estados (0 -> 1 -> 2 -> 6):** Opera de forma análoga às demais instruções do Tipo R. O cálculo aritmético ocorre na ULA durante o Estado 2, onde circuitos dedicados tratam exceções como divisão por zero (retornando -1) ou overflow (retornando o valor de entrada). O resultado final é consolidado e escrito no registrador de destino ao atingir o Estado 6.
