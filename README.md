NARUTO: Ultimate Ninja STORM — Investigação Android

> Investigação, engenharia reversa e estudo técnico da versão Android de NARUTO: Ultimate Ninja STORM.

Status: 🟡 Investigação em andamento, progresso significativo

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

- Versão: 1.2.10
- Distribuição: Google Play Store
- Cópia: oficial e adquirida legalmente
- Arquitetura: ARM64
- Dados do jogo: incluídos no próprio APK

A versão analisada possui os dados do jogo incorporados ao próprio APK. Essa forma de instalação está sendo utilizada porque a inicialização responsável pela instalação ou carregamento dos dados separadamente não está sendo concluída devido ao problema investigado.

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

As ferramentas utilizadas nesta investigação incluem:

- Android: dispositivo utilizado para executar o jogo e realizar os testes.
- Termux: ambiente de terminal utilizado diretamente no Android para executar comandos e analisar a biblioteca nativa.
- LLVM Objdump ("llvm-objdump"): utilizado para desassemblar a biblioteca ARM64 e examinar instruções de máquina.
- GNU "readelf": utilizado para examinar informações do arquivo ELF, símbolos e metadados da biblioteca.
- Ghidra: utilizado anteriormente na análise estática da biblioteca, na identificação de funções e na investigação do fluxo de execução.
- ADB (Android Debug Bridge): utilizado na etapa de extração dos arquivos do jogo e na investigação inicial do problema.
- GitHub: utilizado para documentar publicamente a investigação, registrar o progresso e organizar os resultados.

As ferramentas são empregadas com finalidade educacional, para compreender o comportamento da aplicação e investigar a origem do crash.

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

> Investigação de "pthread"

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

> Progresso da investigação — análise de threads

A investigação avançou para a análise direta do assembly ARM64 da biblioteca "libultimateninjastorm.so", utilizando ferramentas disponíveis no Termux, diretamente em um dispositivo Android.

Além de localizar "mmThread_Impl::WaitForEnd()", foi possível examinar funções relacionadas à criação, à espera e à destruição de threads.

Entre as funções analisadas estão:

- "mmThread_Impl::Start()";
- "nummThreadSimpleThreadCreate()";
- "nummThreadSimpleThreadWaitForEnd()";
- "nuccThread::~nuccThread()".

> Descobertas até o momento

1. Espera pela thread

A função "mmThread_Impl::WaitForEnd()" carrega um valor armazenado em "this + 0x10" e o utiliza na chamada a "pthread_join()".

2. Condição para aguardar a thread

A função "nummThreadSimpleThreadWaitForEnd()" verifica o campo "+0x2c" do contexto. Quando esse campo é zero, retorna sem chamar "WaitForEnd()"; caso contrário, encaminha a execução para essa função.

3. Relação entre criação e espera

A análise de "mmThread_Impl::Start()" revelou uma chamada a "pthread_create()". No caminho de instruções examinado, não foi identificada uma verificação explícita do retorno dessa chamada antes da continuação do fluxo.

Já a função "nummThreadSimpleThreadCreate()" apresenta um comportamento que merece investigação: no caminho analisado, ela retorna "1" e marca o contexto como criado após chamar "Start()".

> Hipótese em investigação

Esses elementos levantam a possibilidade de que uma falha na criação de uma thread não seja devidamente tratada e resulte, posteriormente, em uma tentativa de aguardar uma thread inválida.

Entretanto, essa hipótese ainda não foi confirmada. Não foi demonstrado que "pthread_create()" falha durante a execução do jogo, nem que essa possível falha seja a causa direta do crash.

> Próximos passos

- Analisar "mmRenderManager_Vulkan::Finalize()".
- Verificar a relação entre a finalização do renderizador e as funções de espera identificadas.
- Investigar os caminhos de sucesso e falha de "pthread_create()".
- Determinar se a hipótese proposta corresponde ao caminho real do crash.

Status: análise estática em andamento. A causa raiz ainda não foi identificada e nenhuma correção foi implementada.

---------------------//----------------------

> Investigação adicional — ciclo de vida de "mmThreadHandle"

A análise do assembly ARM64 permitiu aprofundar a relação entre "mmThreadHandle", "mmThread_Impl" e as operações POSIX de gerenciamento de threads.

Descobertas confirmadas

- O construtor de "mmThreadHandle" aloca uma estrutura "mmThread_Impl" e armazena seu ponteiro no início do handle.
- "mmThread_Impl::Start()" passa o endereço do campo "this + 0x10" para "pthread_create()".
- "mmThread_Impl::WaitForEnd()" utiliza o valor armazenado nesse mesmo campo na chamada a "pthread_join()".
- O destrutor de "mmThreadHandle" destrói a implementação e libera sua memória, sem chamar diretamente "pthread_join()".
- "mmRenderManager_Vulkan" mantém dois handles de thread, localizados em "this + 0x3298" e "this + 0x3320". Sua rotina de finalização tenta aguardar as threads associadas a esses handles.

Nova linha de investigação: estado detached

Em "mmThread_Impl::Start()", foi identificada uma ramificação que chama "pthread_attr_setdetachstate()" antes de "pthread_create()".

Ainda é necessário determinar qual parâmetro controla essa ramificação e se ela é utilizada na criação das threads do renderizador Vulkan. Também não foi demonstrado que uma thread criada como detached seja posteriormente passada a "pthread_join()" no caminho que produz o crash.

Status

O ciclo de vida dos handles está parcialmente mapeado. A relação entre o estado da thread, o retorno de "pthread_create()" e a falha observada em "pthread_join()" continua em investigação. A causa raiz ainda não foi confirmada e nenhuma correção foi implementada.

---------------------//----------------------

> Referências e fluxo de execução de "WaitForEnd()"

Durante a análise estática de libultimateninjastorm.so, foram identificadas funções relacionadas à criação, à espera e à destruição de threads.

Funções identificadas:
mmThread_Impl::WaitForEnd() — endereço 0x0abe8cc.
nummThreadSimpleThreadWaitForEnd() — endereço 0x0abe8bc.
mmThread_Impl::Start() — endereço 0x09f6ec4.
nummThreadSimpleThreadCreate() — endereço 0x09f6e28.
nuccThread::~nuccThread() — endereço 0x0964384.

Os endereços correspondem à biblioteca analisada e podem variar entre versões ou compilações diferentes.

> Fluxo de espera

A função nummThreadSimpleThreadWaitForEnd() verifica o campo +0x2c do contexto. Se o valor for zero, retorna sem chamar mmThread_Impl::WaitForEnd(). Caso contrário, encaminha a execução para essa função.

Por sua vez, mmThread_Impl::WaitForEnd() carrega o identificador armazenado em this + 0x10 e o utiliza na chamada a pthread_join().

> Relação com a criação da thread

A função mmThread_Impl::Start() contém uma chamada a pthread_create(), enquanto nummThreadSimpleThreadCreate() apresenta um caminho no qual retorna 1 e marca o contexto como criado após chamar Start().

Essa relação levanta uma hipótese: uma falha na criação da thread poderia não ser devidamente propagada e, posteriormente, resultar em uma tentativa de aguardar uma thread inválida.

Entretanto, ainda não foi demonstrado que essa sequência ocorre durante o crash observado. Também não foi confirmada a relação direta entre esse fluxo e a finalização do renderizador Vulkan.
Próxima investigação

O próximo passo é analisar mmRenderManager_Vulkan::Finalize() e verificar como a finalização do renderizador se relaciona com as funções de espera identificadas.

Status: fluxo parcial mapeado; causa raiz ainda não confirmada.
---------------------//----------------------

> Análise de "mmRenderManager_Vulkan::Finalize()"

Novo resultado da análise estática — 10/10/2026

Foi examinada a implementação de "mmRenderManager_Vulkan::Finalize()" na biblioteca "libultimateninjastorm.so", correspondente ao Build ID associado ao registro de crash de 05/10/2026.

A análise identificou duas chamadas a "mmThread::WaitForEnd(mmThreadHandle&)" durante a finalização do renderizador Vulkan:

- Endereço "0x11ae10c": aguarda a thread associada ao objeto localizado em "this + 0x3298".
- Endereço "0x11ae134": aguarda a thread associada ao objeto localizado em "this + 0x3320".

Antes da primeira espera, a função sinaliza uma condição de sincronização em "this + 0x32A0". Entre as duas esperas, modifica um byte de estado e sinaliza condições associadas a "this + 0x3300" e "this + 0x3308".

Essas instruções demonstram que a finalização do renderizador coordena e aguarda duas threads distintas antes de prosseguir com a liberação dos recursos gráficos.

A relação com o crash é relevante porque a pilha de execução registrada contém "mmRenderManager_Vulkan::Finalize()", "mmThread_Impl::WaitForEnd()" e "pthread_join()", acompanhados da mensagem "invalid pthread_t".

O que ainda não foi determinado

- Qual das duas chamadas a "WaitForEnd()" está associada à falha registrada.
- Como o identificador inválido chegou ao "pthread_join()".
- Se houve falha na criação da thread, corrupção ou sobrescrita do identificador, ou algum problema no ciclo de vida da thread.
- Se a falha é consequência de um erro anterior durante a inicialização do renderizador.

A análise do código não demonstra, por si só, que o Vulkan seja a causa raiz. Também não comprova que a hipótese de falha no tratamento do retorno de "pthread_create()" explique o crash.

Próximos passos

1. Rastrear a inicialização e a criação das threads associadas aos objetos "this + 0x3298" e "this + 0x3320".
2. Identificar os caminhos que inicializam e modificam os respectivos identificadores de thread.
3. Determinar qual chamada de espera corresponde ao caminho da falha.
4. Confrontar os resultados com o backtrace e os metadados do crash registrado.

Status: fluxo de finalização parcialmente mapeado; causa raiz ainda não confirmada; nenhuma correção implementada.

---------------------//----------------------

> Engenharia Reversa

A investigação utiliza técnicas de engenharia reversa para compreender o comportamento interno da versão Android de Naruto Shippuden: Ultimate Ninja Storm e investigar um problema que impede o jogo de iniciar corretamente.

O trabalho concentra-se na análise estática da biblioteca nativa "libultimateninjastorm.so", compilada para a arquitetura ARM64 (AArch64).

> Atividades realizadas

- Extração e identificação dos arquivos da aplicação.
- Análise da estrutura do binário ELF e de sua tabela de símbolos.
- Identificação e desassemblagem de funções relacionadas ao gerenciamento de threads.
- Investigação de "mmThread_Impl::WaitForEnd()" e de sua chamada a "pthread_join()".
- Análise parcial das funções responsáveis pela criação, espera e destruição de threads.
- Exame de trechos do fluxo de execução relacionados ao problema registrado no crash.
- Documentação das descobertas, hipóteses e questões ainda não resolvidas.

> Estado atual

A análise identificou funções e relações relevantes para investigar o erro relacionado a um identificador de thread inválido.

Uma das hipóteses em estudo é que uma falha na criação de uma thread não seja devidamente tratada antes de uma tentativa posterior de aguardar essa thread. Ainda não foi comprovado que essa sequência ocorre durante o crash.

A investigação prossegue com a análise do fluxo de finalização do renderizador Vulkan, buscando estabelecer uma relação entre as funções identificadas e a pilha de execução do crash.

Até o momento, o trabalho consiste principalmente em análise estática. A causa raiz ainda não foi confirmada e nenhuma correção foi implementada.

---------------------//----------------------

> Estudo de ARM64 / Assembly

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

> Metodologia

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

> Objetivos de aprendizado

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

> Roadmap

🟢 Concluído

- [x] Reproduzir o problema
- [x] Capturar o "logcat"
- [x] Identificar o "SIGABRT"
- [x] Identificar a mensagem relacionada a "pthread_join()"
- [x] Localizar "libultimateninjastorm.so"
- [x] Carregar a biblioteca no Ghidra
- [x] Localizar "mmThread_Impl::WaitForEnd()"
- [x] Identificar o acesso a "this + 0x10" em "WaitForEnd()"
- [x] Encontrar referências a "WaitForEnd()"
- [x] Localizar "pthread_create()"
- [x] Identificar o campo utilizado como identificador na chamada a "pthread_join()"
- [x] Confirmar que a biblioteca analisada corresponde ao Build ID de um dos crashes registrados em 05/10/2026
- [x] Localizar "mmThread_Impl::Start()"
- [x] Identificar a chamada a "pthread_create()" dentro de "Start()"
- [x] Identificar a chamada subsequente a "sched_setaffinity()"
- [x] Analisar o construtor "mmThread_ImplC2Ev()" e verificar que ele não apresenta uma escrita explícita em "this + 0x10" no trecho examinado
- [x] Identificar que não há verificação imediata visível do retorno de "pthread_create()" no trecho analisado de "Start()"
- [x] Localizar e analisar "mmThread_Impl::_Copy(...)"
- [x] Confirmar que "_Copy(...)" copia o campo "this + 0x10" do objeto de origem para o objeto de destino
- [x] Localizar e analisar "mmThread_Impl::SetAffinity(...)"
- [x] Confirmar que "SetAffinity()" utiliza os 32 bits inferiores de "this + 0x10" na chamada a "sched_setaffinity()"
- [x] Analisar "mmRenderManager_Vulkan::Finalize()"
- [x] Identificar as duas chamadas a "mmThread::WaitForEnd()" em "Finalize()"
- [x] Identificar os handles associados às esperas, em "this + 0x3298" e "this + 0x3320"
- [x] Localizar o construtor de "mmThreadHandle"
- [x] Confirmar que o construtor aloca uma estrutura "mmThread_Impl" e armazena seu ponteiro no handle
- [x] Analisar o destrutor de "mmThreadHandle" e confirmar que ele destrói e libera a implementação associada
- [x] Analisar o destrutor de "mmThread_Impl" e verificar que ele não chama diretamente "pthread_join()" no trecho examinado
- [x] Identificar que "mmThread_Impl::Start()" possui uma ramificação que configura o estado da thread por meio de "pthread_attr_setdetachstate()"

🟡 Em andamento

- [ ] Rastrear o parâmetro que controla a ramificação de "pthread_attr_setdetachstate()" em "Start()"
- [ ] Verificar se as threads do renderizador Vulkan são criadas como joinable ou detached
- [ ] Investigar o tratamento dos retornos de "pthread_create()" e das funções de configuração dos atributos
- [ ] Verificar se o identificador em "this + 0x10" é inicializado ou alterado em outros pontos
- [ ] Investigar se o valor armazenado em "this + 0x10" pode ser utilizado corretamente nas chamadas a "pthread_join()" e "sched_setaffinity()"
- [ ] Investigar se copiar o campo "this + 0x10" entre objetos pode afetar o ciclo de vida das threads
- [ ] Compreender o ciclo de vida das threads: criação, espera e destruição
- [ ] Relacionar as funções de gerenciamento de threads à finalização do renderizador Vulkan
- [ ] Investigar a relação entre "pthread_create()", "sched_setaffinity()" e a utilização posterior do identificador
- [ ] Determinar qual das duas esperas em "Finalize()" está relacionada ao crash histórico
- [ ] Determinar por que o identificador passado a "pthread_join()" é considerado inválido
- [ ] Identificar e comprovar a causa raiz do crash
- [ ] Avaliar hipóteses de correção com base nas evidências obtidas

🔴 Futuro

- [ ] Desenvolver uma possível correção
- [ ] Testar a correção em ambiente controlado
- [ ] Documentar o processo completo e as evidências
- [ ] Criar ferramentas auxiliares próprias para análise
- [ ] Investigar outras incompatibilidades com versões modernas do Android
- [ ] Avaliar possibilidades de manutenção e compatibilidade do aplicativo

---------------------//----------------------

> Contribuições

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

> Aviso legal

Este é um projeto independente, sem afiliação, associação ou endosso da Bandai Namco Entertainment, CyberConnect2 ou dos respectivos detentores de direitos.

Os direitos autorais referentes ao jogo, seu código, bibliotecas, personagens, imagens, músicas, marcas e demais conteúdos permanecem pertencendo aos seus respectivos detentores.

Este repositório não tem como objetivo redistribuir arquivos proprietários do jogo.

Arquivos como APKs, bibliotecas nativas, assets ou outros componentes proprietários não são disponibilizados neste repositório.

A análise está utilizando uma cópia obtida legalmente e arquivos extraídos localmente pelo pesquisador.

---------------------//----------------------

> Licença

Os textos, documentação, scripts e demais materiais originais criados especificamente para este projeto são disponibilizados sob a licença MIT, salvo indicação em contrário.

A licença MIT deste repositório não concede qualquer direito sobre o jogo NARUTO: Ultimate Ninja STORM ou sobre seus componentes proprietários.

---------------------//----------------------

Enfim, como podem ver, este projeto de estudo começou com um jogo que não inicia.

A intenção é terminar com uma compreensão muito maior de programação, sistemas, Android, engenharia reversa, debugging e desenvolvimento de software.

E, se no caminho for possível descobrir uma maneira de fazê-lo funcionar novamente, melhor ainda.

Muito obrigado pela atenção e peço paciência. Vou atualizando o repositório conforme os progressos forem surgindo.
