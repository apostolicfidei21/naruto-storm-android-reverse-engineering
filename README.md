NARUTO: Ultimate Ninja STORM — Investigação Android

> Investigação, engenharia reversa e estudo técnico da versão Android de NARUTO: Ultimate Ninja STORM.

Status: 🟡 Investigação em andamento, fase inicial

---------------------//----------------------

> Sobre o projeto

Sou estudante de Cibersegurança e, como parte dos meus estudos pessoais, decidi colocar meus conhecimentos em prática através de um problema real, utilizando a investigação como uma forma de aprendizado.

NARUTO: Ultimate Ninja STORM é um jogo pelo qual tenho grande apreço. Adquiri a versão para Android quando foi lançada e cheguei a concluí-la. Entretanto, desde o lançamento, o jogo apresentou diversos problemas, incluindo falhas aleatórias e dificuldades relacionadas ao salvamento automático.

Com o passar do tempo, tive o desejo de revisitá-lo, porém o aplicativo deixou de iniciar completamente. Atualmente, ao tentar executá-lo, o jogo encerra durante a inicialização e o Android informa que o aplicativo possui um bug.

O problema persiste mesmo após tentativas com versões anteriores do aplicativo, o que despertou meu interesse em investigar tecnicamente a causa da falha.

Assim, este projeto nasceu da tentativa de descobrir por que a versão Android de NARUTO: Ultimate Ninja STORM deixou de iniciar corretamente em dispositivos Android modernos.

Em vez de simplesmente tentar encontrar e publicar uma solução pronta, a proposta é investigar tecnicamente o problema, entender o funcionamento do jogo e utilizar todo o processo como um projeto prático de estudo em programação, sistemas, engenharia reversa e desenvolvimento multiplataforma. Tive também a boa vontade de compartilhar meu progresso, a fim de que isso também possa servir como material de estudo para outras pessoas.

A problemática central do projeto é: entender por que o jogo não funciona e se uma solução é possível.

---------------------//----------------------

> Ambiente de Teste e Versão Analisada

A investigação está sendo realizada utilizando uma cópia oficial e legalmente adquirida de NARUTO: Ultimate Ninja STORM, obtida através da Google Play Store.

Versão do jogo analisada:

- Versão: 1.2.9
- Distribuição: Google Play Store
- Cópia: oficial e adquirida legalmente
- Arquitetura: ARM64
- Dados do jogo: incluídos no próprio APK

A versão analisada possui os dados do jogo incorporados ao próprio APK. Essa forma de instalação está sendo utilizada porque a inicialização responsável pela instalação ou carregamento dos dados separadamente não está sendo concluída devido ao problema investigado.

//OBS.: o jogo atualizou recentemente para a versão 1.2.10, mas continuo usando a versão anterior, uma vez que o problema persiste independente da versão e seria muito trabalhoso replicar toda a data novamente cada vez que o jogo atualizasse, então preferi essa modo de proceder. Se conseguir encontrar uma solução para esta versão analisada, é possível que ela seja replicada em outras posteriores//

Dispositivo principal de teste:

- Samsung Galaxy S24 FE
- Android 15
- ARM64 / AArch64
- Samsung Exynos 2400e
- GPU Samsung Xclipse 940
- 8 GB de RAM
- Vulkan 1.3
- One UI 7

O objetivo é documentar e reproduzir o comportamento dessa versão específica do aplicativo em um ambiente Android moderno, mantendo as condições de teste o mais consistentes possível.

---------------------//----------------------

> Ferramentas e Tecnologias

As ferramentas abaixo estão sendo utilizadas ou serão utilizadas ao longo da investigação, de acordo com as necessidades de cada etapa.

Ferramentas já utilizadas

- ADB (Android Debug Bridge) — comunicação com o dispositivo Android, coleta de informações e obtenção de arquivos do aplicativo.
- Android SDK Platform-Tools — conjunto de ferramentas utilizado principalmente para executar comandos ADB.
- Ghidra — análise estática da biblioteca nativa, incluindo desmontagem, decompilação, análise de referências e investigação do fluxo de execução.
- Android Logcat — captura e análise dos logs gerados pelo sistema e pelo aplicativo durante a inicialização e o crash.

Ferramentas que poderão ser utilizadas

- LLDB — debugging de código nativo e análise da execução em tempo real.
- Android NDK — estudo e desenvolvimento de código nativo para Android.
- IDA / ferramentas alternativas de engenharia reversa — eventualmente utilizadas para comparação de resultados com o Ghidra.
- Ferramentas de análise ELF — inspeção das bibliotecas nativas e de suas estruturas.
- Ferramentas de compilação C/C++ — desenvolvimento de testes e possíveis ferramentas auxiliares.
- Git / GitHub — versionamento e documentação da investigação.

A utilização de cada ferramenta dependerá da evolução da investigação, com outras podendo ainda ser adicionadas. O objetivo é utilizar o mínimo necessário em cada etapa e compreender o papel de cada ferramenta no processo, em vez de apenas aplicá-las como uma "caixa-preta".

---------------------//----------------------

> Problema inicial

Ao tentar iniciar o jogo em um dispositivo Android moderno, ocorre um encerramento durante a inicialização.

A análise inicial através do "logcat" indicou um crash nativo envolvendo:

- "SIGABRT" (signal 6);
- "pthread_join()";
- mensagem relacionada a "invalid pthread_t";
- "libultimateninjastorm.so";
- inicialização/finalização do sistema de renderização;
- componentes relacionados ao Vulkan.

Um dos pontos mais importantes encontrados foi:

invalid pthread_t ... passed to pthread_join

Isso indica que o programa tentou utilizar "pthread_join()" com um identificador de thread inválido.

Ainda não está determinado se esse é o problema original ou apenas a consequência final de uma falha anterior.

---------------------//----------------------

> Análise do crash

A análise do "logcat" permitiu identificar uma sequência aproximada de chamadas:

androidMain()
 └── libnucc_Init()
      └── nuccSys::Initialize()
           └── nuccSys::InitRender()
                └── nummRender::Init()
                     └── mmPlatformLibrary::Initialize()
                          └── mmDeviceManager::Initialize()
                               └── mmGPUDevice_Android::Initialize()
                                    └── mmGPUDevice_Android::InitializeRenderSystem()
                                         └── mmGPUDevice_Android::Finalize()
                                              └── mmRenderManager_Vulkan::~mmRenderManager_Vulkan()
                                                   └── mmRenderManager_Vulkan::Finalize()
                                                        └── mmThread_Impl::WaitForEnd()
                                                             └── pthread_join()
                                                                  └── SIGABRT

Essa sequência ainda está sendo investigada e não deve ser considerada uma prova definitiva da causa raiz.

O fato de o crash aparecer durante um caminho relacionado ao Vulkan não significa necessariamente que o Vulkan seja o responsável direto pelo problema.

Possibilidades ainda em investigação incluem:

- falha na criação de uma thread;
- "pthread_t" não inicializado;
- identificador de thread sobrescrito;
- tentativa de finalizar uma thread que nunca foi criada corretamente;
- tentativa de aguardar uma thread já finalizada ou invalidada;
- falha parcial durante a inicialização do sistema de renderização;
- incompatibilidade entre código nativo antigo e versões modernas do Android;
- outros problemas de inicialização que acabam levando ao "pthread_join()" inválido.

---------------------//----------------------

Investigação de "pthread"

Um dos principais pontos encontrados foi a função:

- mmThread_Impl::WaitForEnd()

A análise no Ghidra revelou uma chamada equivalente a:

- pthread_join(*(pthread_t *)(this + 0x10), &local_20);

Em termos simplificados, o código está obtendo um valor localizado no objeto em:

- this + 0x10

tratando esse valor como um "pthread_t" e passando-o para:

- pthread_join()

A investigação agora busca descobrir:

1. Onde esse "pthread_t" é criado;
2. Onde o valor é armazenado em "this + 0x10";
3. Se "pthread_create()" foi executado corretamente;
4. Se a criação da thread falha em determinadas condições;
5. Quem chama "WaitForEnd()";
6. Em que momento a thread é finalizada;
7. Por que o identificador chega inválido ao "pthread_join()".

---------------------//----------------------

Referências a "WaitForEnd()"

A análise de referências no Ghidra encontrou chamadas para:

mmThread_Impl::WaitForEnd()

Entre elas:

00bc50c4
010b5d98
010b5df4
0128262c

As referências:

010b5d98
010b5df4
0128262c

são atualmente alguns dos principais pontos de interesse, pois podem representar chamadas externas à função durante diferentes caminhos de execução.

O próximo objetivo é analisar cada uma dessas funções e determinar qual delas está relacionada ao caminho de finalização do renderer Vulkan.

---------------------//----------------------

Engenharia reversa

A investigação está sendo realizada principalmente através do Ghidra, analisando a biblioteca nativa:

libultimateninjastorm.so

O arquivo é uma biblioteca ELF ARM64 utilizada pelo aplicativo Android.

Entre os elementos analisados estão:

- ".text";
- ".rodata";
- ".data";
- ".bss";
- ".dynsym";
- ".dynstr";
- ".rela.dyn";
- ".rela.plt";
- ".got";
- ".plt";
- informações de unwind;
- símbolos e referências;
- código ARM64/AArch64;
- chamadas para funções externas.

Os arquivos originais do jogo não fazem parte deste repositório.

---------------------//----------------------

Estudo de ARM64 / Assembly

Além da investigação específica do crash, o projeto também serve como estudo prático de assembly ARM64.

Alguns conceitos estudados:

- registradores "X0–X30";
- "SP";
- "PC";
- "LDR" / "STR";
- "ADR" / "ADRP";
- "ADD" / "SUB";
- "BL";
- "BR";
- offsets de estruturas;
- chamadas de função;
- convenção de chamada AArch64;
- ponteiros;
- referências;
- thunks;
- branches;
- funções externas.

Um dos objetivos é compreender o código descompilado não apenas como "código que o Ghidra mostrou", mas relacioná-lo ao assembly original.

---------------------//----------------------

Metodologia

A investigação segue, sempre que possível, uma abordagem incremental:

Crash
  ↓
Logcat
  ↓
Backtrace
  ↓
Função relacionada
  ↓
Código nativo
  ↓
Assembly / Ghidra
  ↓
Referências
  ↓
Chamadores
  ↓
Origem dos dados
  ↓
Causa provável
  ↓
Teste de hipótese
  ↓
Possível correção

A intenção é evitar alterações aleatórias no aplicativo antes de compreender o problema.

---------------------//----------------------

Objetivos de aprendizado

Este projeto também funciona como um laboratório pessoal de estudos.

Programação

- C;
- C++;
- ponteiros;
- estruturas;
- memória;
- orientação a objetos;
- threads;
- gerenciamento de recursos;
- programação de sistemas.

Android

- arquitetura Android;
- APKs;
- bibliotecas nativas;
- JNI;
- NDK;
- ABI;
- ARM64;
- processos e threads;
- "logcat";
- ADB.

Engenharia reversa

- Ghidra;
- ELF;
- desmontagem;
- decompilação;
- assembly;
- símbolos;
- referências;
- chamadas de função;
- análise de fluxo;
- análise de crash.

Sistemas

- POSIX;
- "pthread";
- "pthread_create()";
- "pthread_join()";
- sincronização;
- ciclo de vida de threads;
- processos;
- memória.

Gráficos

- Vulkan;
- inicialização de renderizadores;
- gerenciamento de recursos;
- ciclo de vida de dispositivos gráficos.

Segurança e pesquisa

- análise de binários;
- debugging;
- análise de vulnerabilidades;
- investigação sistemática;
- documentação técnica;
- reprodução de problemas.

---------------------//----------------------

Roadmap

🟢 Concluído

- [x] Reproduzir o problema
- [x] Capturar o "logcat"
- [x] Identificar o "SIGABRT"
- [x] Identificar a mensagem relacionada a "pthread_join()"
- [x] Localizar "libultimateninjastorm.so"
- [x] Carregar a biblioteca no Ghidra
- [x] Localizar "mmThread_Impl::WaitForEnd()"
- [x] Identificar o acesso a "this + 0x10"
- [x] Encontrar referências a "WaitForEnd()"

🟡 Em andamento

- [ ] Analisar os principais chamadores de "WaitForEnd()"
- [ ] Identificar a estrutura de "mmThread_Impl"
- [ ] Localizar "pthread_create()"
- [ ] Descobrir onde o "pthread_t" é armazenado
- [ ] Determinar por que o identificador se torna inválido
- [ ] Identificar a causa raiz do crash
- [ ] Testar hipóteses de correção

🔴 Futuro

- [ ] Desenvolver uma possível correção
- [ ] Testar a correção em ambiente controlado
- [ ] Documentar o processo completo
- [ ] Criar ferramentas auxiliares próprias
- [ ] Investigar outras incompatibilidades com Android moderno
- [ ] Avaliar possibilidades de manutenção/compatibilidade do aplicativo

---------------------//----------------------

Contribuições

Este projeto está aberto a sugestões, correções e discussões técnicas.

Se você possui conhecimento sobre:

- C/C++;
- ARM64;
- Android NDK;
- pthreads;
- Vulkan;
- Ghidra;
- engenharia reversa;
- debugging;
- desenvolvimento de jogos;

Sugestões e explicações são muito bem-vindas.

O objetivo é aprender através da investigação e, se possível, descobrir uma solução que também possa ser útil para outras pessoas.

---------------------//----------------------

Aviso legal

Este é um projeto independente, sem afiliação, associação ou endosso da Bandai Namco Entertainment, CyberConnect2 ou dos respectivos detentores de direitos.

Os direitos autorais referentes ao jogo, seu código, bibliotecas, personagens, imagens, músicas, marcas e demais conteúdos permanecem pertencendo aos seus respectivos detentores.

Este repositório não tem como objetivo redistribuir arquivos proprietários do jogo.

Arquivos como APKs, bibliotecas nativas, assets ou outros componentes proprietários não são disponibilizados neste repositório.

A análise está utilizando uma cópia obtida legalmente e arquivos extraídos localmente pelo pesquisador.

---------------------//----------------------

Licença

Os textos, documentação, scripts e demais materiais originais criados especificamente para este projeto são disponibilizados sob a licença MIT, salvo indicação em contrário.

A licença MIT deste repositório não concede qualquer direito sobre o jogo NARUTO: Ultimate Ninja STORM ou sobre seus componentes proprietários.

---------------------//----------------------

Filosofia do projeto

> O objetivo não é apenas fazer o jogo funcionar. O objetivo é entender por que ele não funciona.

Este projeto começou com um jogo que não inicia.

A intenção é terminar com uma compreensão muito maior de programação, sistemas, Android, engenharia reversa, debugging e desenvolvimento de software.

E, se no caminho for possível descobrir uma maneira de fazê-lo funcionar novamente, melhor ainda.

Muito obrigado pela atenção e peço paciência. Vou atualizando o repositório conforme os progressos forem surgindo.