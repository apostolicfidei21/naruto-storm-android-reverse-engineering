NARUTO: Ultimate Ninja STORM — Investigação Android

> Investigação, engenharia reversa e estudo técnico da versão Android de NARUTO: Ultimate Ninja STORM.

Status: 🟡 Causa imediata identificada; causa raiz ("qual chamada Vulkan falha?") em investigação.

'OBS: As datas ao longo deste arquivo registram o andamento da investigação,não necessariamente o momento em que o defeito surgiu no código ou no ambiente.'

Atualização - dia 11 de outubro de 2026:
- seção PROBLEMA INICIAL removida [estava desatualizada];
- seção AMBIENTE DE TESTE E VERSÃO ANALISADA atualizada (Versão do Android e One UI atualizadas, adicionado versão do driver gráfico e Build ID da biblioteca nativa do jogo);
- seções com suposições/perguntas já esclarecidas foram removidas;
- novas seções com evidências confirmadas foram adicionadas;
- seção de RESUMO disponível.

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
- Biblioteca nativa: libultimateninjastorm.so, arquitetura ARM64, Build ID c3e506f9be350dfb4a00c79228754028
- Instalação: APK base + splits (InstallPack, config.arm64_v8a, idioma e densidade)
- targetSdk: 36

Histórico: o jogo foi comprado e jogado no lançamento, em 24 de setembro de 2024, quando o aparelho utilizava Android 14 / One UI 6.1 e o jogo funcionava. Essa diferença de ambiente é relevante para a investigação, mas, isoladamente, não prova que uma atualização do Android ou do driver tenha causado o problema.

O Build ID identifica a compilação específica da biblioteca nativa examinada. Ele ajuda a distinguir essa versão de outras compilações de libultimateninjastorm.so, evitando que endereços encontrados nesta análise sejam tratados como universais.

Dispositivo principal de teste:

- Samsung Galaxy S24 FE
- Android 16
- ARM64 / AArch64
- Samsung Exynos 2400e
- GPU Samsung Xclipse 940
- Driver Vulkan: Samsung XGL/SUMD 24.0.534, git 1900168dcb, compilado em 09/01/2026
- 8 GB de RAM
- Vulkan 1.3
- One UI 8.5

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

> Resumo

Sintoma [confirmado]: o jogo encerra durante a inicialização e o Android registra SIGABRT. Esse sinal indica que o processo foi abortado; neste caso, o log aponta para uma chamada inválida a pthread_join().

Causa imediata [confirmada pela análise estática e pelo log]: durante a inicialização do renderizador Vulkan, ocorre uma exceção C++ — provavelmente uma vk::*Error, usada pelo Vulkan-Hpp para representar erros da API Vulkan. A exceção é capturada por um catch em mmGPUDevice_Android::InitializeRenderSystem(), que chama Finalize() para limpar os recursos. O problema é que essa limpeza tenta executar pthread_join() usando um handle de thread que ainda não foi inicializado corretamente. A biblioteca Bionic, responsável pelas funções do sistema C no Android, detecta o identificador inválido e aborta o processo.

Causa raiz [em investigação]: ainda não foi identificado qual chamada Vulkan retorna o erro que provoca a exceção, nem por que isso acontece. As possibilidades investigadas e o estado das evidências estão reunidos na seção Hipóteses e status.

Limitações da análise: até aqui, a investigação é essencialmente estática: o código nativo foi examinado no Ghidra e via Termux, mas ainda não houve execução instrumentada para observar diretamente os retornos das chamadas Vulkan. Por isso, os endereços e offsets apresentados correspondem exclusivamente à biblioteca analisada e podem mudar em outras versões do jogo.

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

---------------------//----------------------

> Progresso da investigação — análise de threads - 7 de outubro de 2026

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

> Investigação adicional — ciclo de vida de "mmThreadHandle" - 8 de outubro de 2026

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

Status: o ciclo de vida dos handles está parcialmente mapeado. A relação entre o estado da thread, o retorno de "pthread_create()" e a falha observada em "pthread_join()" continua em investigação. A causa raiz ainda não foi confirmada e nenhuma correção foi implementada.

---------------------//----------------------

> Referências e fluxo de execução de "WaitForEnd()" - 9 de outubro de 2026

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

> Análise de "mmRenderManager_Vulkan::Finalize()" — 10 de outubro de 2026

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

Resultados recentes: falha na inicialização gráfica - 10 de outubro de 2026

Evidência em tempo de execução

Durante a inicialização do jogo, foi registrado o seguinte erro fatal:

"invalid pthread_t 0x2e79616c702e6469 passed to pthread_join"

O processo foi encerrado por "SIGABRT". O backtrace identifica a seguinte cadeia de chamadas:

"pthread_join()"
→ "mmThread_Impl::WaitForEnd()"
→ "mmRenderManager_Vulkan::Finalize()"
→ "mmRenderManager_Vulkan::~mmRenderManager_Vulkan()"
→ "mmGPUDevice_Android::Finalize()"
→ "mmGPUDevice_Android::InitializeRenderSystem()"

O ponto de falha em "Finalize()" corresponde à primeira chamada a "WaitForEnd()" nessa rotina, relacionada ao primeiro identificador de thread do renderizador Vulkan.

Resultados da análise estática

- "mmThread_Impl::WaitForEnd()" carrega o identificador de thread do objeto e o encaminha para "pthread_join()" sem uma verificação de validade visível nesse trecho.
- "mmThread_Impl::Start()" chama "pthread_create()", mas não verifica explicitamente seu código de retorno antes de continuar a execução.
- O endereço de retorno "0xD37A10" pertence a um caminho de tratamento de exceção em "mmGPUDevice_Android::InitializeRenderSystem()", que executa uma rotina de limpeza antes de retornar um erro.

Hipóteses ainda em investigação

1. Falha na criação de uma thread seguida de uma tentativa de aguardá-la.
2. Inconsistência no estado do renderizador durante a limpeza após uma exceção.
3. Corrupção ou sobrescrita do identificador armazenado no objeto de thread.

Essas hipóteses ainda não foram confirmadas. A exceção que inicia a limpeza e a origem exata do identificador inválido continuam desconhecidas.

Próximos passos

- Identificar a operação que inicia o caminho de tratamento de exceção.
- Verificar a sequência de inicialização das threads do renderizador Vulkan.
- Comparar os diferentes pontos de falha observados nos logs.
- Só então avaliar uma possível correção e sua viabilidade.

Conclusão provisória: a falha foi localizada no encerramento de uma thread durante a limpeza da inicialização gráfica. Isso identifica o caminho imediato do crash, mas não estabelece ainda sua causa original.

---------------------//----------------------

> Evidência Recente: o pthread_t contém resíduos de texto, não um endereço de thread válido - 11 de outubro de 2026

Um logcat recente, realizado na noite passada via Termux ADB, registrou dois valores inválidos passados a pthread_join(). Quando esses valores são interpretados como bytes em little-endian — formato no qual o byte menos significativo aparece primeiro —, eles correspondem aos seguintes textos ASCII:

| Valor | Bytes em little-endian | Texto ASCII |
|---|---|---|
| `0x2e79616c702e6469` | `69 64 2e 70 6c 61 79 2e` | `id.play.` |
| `0x2f79616c702f6469` | `69 64 2f 70 6c 61 79 2f` | `id/play/` |

Esses valores não se parecem com identificadores de threads válidos. Sua representação em ASCII sugere que a memória contém resíduos de texto, possivelmente partes de nomes de classes ou caminhos relacionados ao Play Core, mecanismo do Android utilizado, entre outras funções, na distribuição de recursos adicionais por asset packs.

Há também uma coincidência temporal relevante: no log, a mesma thread identificada como androidMain registra um listener do Play Core poucos milissegundos antes do crash. Isso torna a relação com os resíduos de memória plausível, mas não comprova que o Play Core tenha causado a falha.

Minha Interpretação (alta confiança, ainda não verificada em tempo de execução): o campo localizado em this + 0x10 de mmThread_Impl provavelmente não foi inicializado. Em C++, this aponta para o objeto atual, e +0x10 indica um deslocamento de 16 bytes a partir do início desse objeto.

A análise do construtor de mmThreadHandle indica que o objeto é alocado sem que esse campo receba um valor inicial. Se nenhuma escrita posterior preencher o campo, ele poderá conservar o conteúdo que já existia naquela região da memória. Esse conteúdo residual, conhecido informalmente como heap garbage, explicaria por que um texto ASCII aparece onde deveria haver um handle de thread.

Essa interpretação é compatível com a ausência de uma escrita em this + 0x10 no construtor analisado. Entretanto, a confirmação definitiva exige verificar o comportamento em execução e descartar possíveis escritas indiretas no campo.

> InitializeRenderSystem: como a exceção chega ao Finalize()

A função mmGPUDevice_Android::InitializeRenderSystem(), localizada em 0xd3783c, é responsável por inicializar os componentes necessários ao sistema de renderização. A análise da região de tratamento de exceções mostra um bloco try em 0xd378a8–0xd37944 e um landing pad em 0xd379f0.

Em C++, um bloco try delimita operações cujas exceções podem ser tratadas por um catch. O landing pad é o ponto de entrada do código gerado para encaminhar a execução ao tratamento apropriado quando uma exceção é lançada.

A tabela de exceções, conhecida como LSDA (Language-Specific Data Area), indica que essa região cobre exatamente quatro chamadas:

1. mmRenderManager_Vulkan::CreateVulkanInstance() — cria a instância Vulkan, que representa o ponto inicial de acesso à API Vulkan pelo aplicativo.

2. mmGPUDevice_Android::InitializeVulkanDevice() — inicializa o dispositivo Vulkan utilizado para executar as operações gráficas.

3. vmaCreateAllocator — cria um alocador de memória com a Vulkan Memory Allocator (VMA), biblioteca auxiliar usada para gerenciar alocações de memória Vulkan.

4. mmRenderManager_Vulkan::Initialize(mmDeviceDesc const&) — inicializa o gerenciador de renderização Vulkan, incluindo os recursos necessários para produzir imagens.

Achado (confirmado por análise estática): os retornos dessas chamadas não são verificados explicitamente nesse trecho. Isso não significa que nenhuma delas possa detectar erros: wrappers do Vulkan-Hpp podem lançar exceções quando uma operação retorna um código de erro.

A biblioteca contém informações de tipo (typeinfo) para vk::SystemError e suas subclasses vk::*Error. Esses tipos pertencem ao mecanismo de exceções do Vulkan-Hpp, uma interface C++ para Vulkan. Os wrappers examinados possuem blocos de código que lançam exceções para códigos de erro específicos. Assim, um VkResult de erro pode ser convertido em uma exceção C++, interrompendo o fluxo normal de inicialização.

O problema é que o tratamento observado não registra a mensagem original da exceção. Em vez disso, o handler executa a seguinte sequência:

1. __cxa_begin_catch: inicia o tratamento da exceção C++.

2. Finalize(): chama a rotina de finalização e limpeza, por meio da vtable no deslocamento +0x20.

3. __cxa_end_catch: encerra o tratamento da exceção.

4. Retorna 0x80000000, valor utilizado nesse caminho de falha.

A vtable, ou tabela virtual, contém os endereços das funções virtuais de uma classe. O deslocamento +0x20 identifica a entrada utilizada para chegar à função de finalização nesse contexto.

O endereço de retorno 0xD37A10, identificado no backtrace, corresponde à chamada a Finalize() dentro do catch. Isso conecta a análise estática ao caminho de execução registrado no crash: a inicialização falha, a exceção é capturada e a limpeza é iniciada.

A investigação indica, portanto, que existem dois eventos encadeados: primeiro, uma exceção durante a inicialização Vulkan; depois, uma falha na limpeza ao tentar aguardar uma thread que ainda não foi iniciada.

O que ainda falta descobrir: qual das quatro chamadas origina a exceção e qual código de erro Vulkan é retornado. A presença dos tipos vk::*Error sustenta a interpretação de uma exceção Vulkan, mas, sem instrumentação em execução ou evidência equivalente, não identifica a chamada específica que falhou.

> Ordem de criação das threads

Para entender por que pthread_join() recebe um handle inválido, é necessário verificar quando as threads são criadas em relação às operações Vulkan.

A função mmRenderManager_Vulkan::Initialize, localizada em 0x11ae4c0, executa a inicialização dos recursos gráficos. A análise estática indica que as duas threads do renderizador só são iniciadas no final dessa sequência, depois da criação dos objetos Vulkan.

A ordem observada é:

CreateSurface → CreateSwapchain → InitializeMainThread → CreateThreadLocal → CreateDepthBuffer → CreateRenderPass → CreateFramebuffer → CreateCommandBuffer → InitializeSwapchainImageLayout → InitializeGraphicsPipeline → mmThread::Start → mmThread::Start.

As operações anteriores a mmThread::Start preparam a superfície de apresentação, o swapchain e os recursos necessários à renderização. Em seguida, as chamadas a Start iniciam as threads do renderizador.

Os dois pontos de início identificados são:

• Primeira thread: handle em this + 0x3298, chamada a mmThread::Start em 0x11ae75c.

• Segunda thread: handle em this + 0x3320, chamada a mmThread::Start em 0x11ae79c.

A sequência é importante porque o backtrace aponta para o primeiro WaitForEnd(), associado ao handle em +0x3298. Se o fluxo de execução chegou ao catch antes de alcançar 0x11ae75c, a primeira thread do renderizador ainda não havia sido iniciada.

Conclusão (confirmada pela análise estática da ordem de execução): a hipótese de que pthread_create() falhou ao tentar criar essa thread não explica o crash observado. A chamada a Start fica depois das operações que podem provocar a exceção; portanto, se a exceção interrompe a inicialização antes desse ponto, a criação da thread nem sequer é tentada.

Isso também muda a interpretação de pthread_join(): o identificador inválido não precisa ter sido corrompido durante uma tentativa de criação. Ele pode simplesmente nunca ter recebido um valor válido antes de a rotina de limpeza tentar utilizá-lo.

Essa conclusão diz respeito ao caminho de falha analisado. Não demonstra que todas as chamadas pthread_create() existentes no aplicativo sejam bem-sucedidas em qualquer circunstâncias.

---------------------//----------------------

> Parâmetros do swapchain (CreateSwapchain, 0x11ae91c) - 11 de outubro de 2026

O swapchain é a estrutura que reúne as imagens utilizadas para apresentar os quadros do jogo na tela. Sua criação depende das capacidades da superfície Android e dos recursos oferecidos pelo dispositivo Vulkan e pelo driver gráfico.

A função CreateSwapchain, localizada em 0x11ae91c, consulta os formatos e as capacidades da superfície e, posteriormente, chama vkCreateSwapchainKHR em 0x11aebec.

A tabela a seguir reúne os parâmetros identificados por meio do layout da pilha e de literais encontrados em .rodata, a seção da biblioteca que armazena dados constantes. Esses valores são achados da análise estática e ainda devem ser conferidos no Ghidra.

| Campo | Valor identificado | O que significa |
|---|---|---|
| `imageFormat` / `imageColorSpace` | Primeiro item de `vkGetPhysicalDeviceSurfaceFormatsKHR` | O formato de pixel e o espaço de cores usados pelas imagens do swapchain são obtidos do primeiro formato retornado pela consulta à superfície. |
| `minImageCount` | `minImageCount + 1`, limitado ao máximo | O código solicita uma imagem a mais que o mínimo exigido, respeitando o limite máximo quando ele é diferente de zero. |
| `imageExtent` | `mmCoreLibrary::s_deviceWidth × s_deviceHeight` (16 bits) | Define a largura e a altura das imagens a partir das dimensões armazenadas pelo jogo, em vez de utilizar diretamente `currentExtent`. |
| `imageArrayLayers` | `1` | Cada imagem do swapchain tem uma camada. |
| `imageUsage` | `19 = COLOR_ATTACHMENT + TRANSFER_SRC + TRANSFER_DST` (literal em `0x66c9a0`) | Declara que as imagens serão utilizadas como destinos de renderização e em operações de transferência de dados. |
| `preTransform` | `1 = IDENTITY` (literal em `0x66ea10`) | Solicita que as imagens sejam apresentadas sem aplicar uma transformação de orientação à imagem. |
| `compositeAlpha` | `8 = INHERIT` | Seleciona o modo de composição alfa `INHERIT`, no qual o comportamento do canal alfa é herdado do sistema de composição. |
| `presentMode` | `2 = FIFO` | Seleciona o modo de apresentação FIFO, no qual as imagens são apresentadas em uma fila, em vez de substituir livremente a imagem pendente. |
| `clipped` | `1` | Permite que o conteúdo de pixels encobertos por outras regiões seja tratado como não relevante para a apresentação. |

O valor imageUsage = 19 corresponde à combinação dos seguintes indicadores Vulkan:

• VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT (16): permite usar a imagem como destino de operações de renderização.

• VK_IMAGE_USAGE_TRANSFER_SRC_BIT (1): permite usar a imagem como origem de uma operação de transferência.

• VK_IMAGE_USAGE_TRANSFER_DST_BIT (2): permite usar a imagem como destino de uma operação de transferência.

A soma dos indicadores é 16 + 1 + 2 = 19.

Possível ponto de incompatibilidade [hipótese]: o código parece consultar as capacidades da superfície, mas utiliza apenas minImageCount e maxImageCount na seleção dos parâmetros descritos. Não foi identificada, nesse trecho, uma verificação equivalente para supportedUsageFlags, supportedCompositeAlpha, supportedTransforms, currentTransform ou currentExtent.

Esses campos são importantes porque descrevem restrições e propriedades da superfície. Por exemplo, supportedUsageFlags indica quais usos de imagem são permitidos; supportedCompositeAlpha informa os modos de composição alfa disponíveis; e supportedTransforms lista as transformações suportadas.

O campo currentTransform descreve a transformação atual da superfície, enquanto currentExtent informa a extensão atual recomendada ou definida para as imagens do swapchain, conforme a plataforma e o driver. Utilizar IDENTITY e dimensões próprias, sem considerar adequadamente essas propriedades, pode ser incompatível com determinados ambientes. Isso, porém, ainda não foi demonstrado como a causa deste crash.

As dimensões usadas pelo jogo vêm de mmCoreLibrary::s_deviceWidth (0x15654d0) e mmCoreLibrary::s_deviceHeight (0x15654d4). Esses valores são gravados em OnAppCmd, localizada em 0x1151858, a partir de chamadas a ANativeWindow_getWidth/Height, que consultam a largura e a altura da janela nativa Android.

A mesma função contém uma chamada a pthread_create() em 0x1151a10, possivelmente relacionada ao início da thread androidMain. Essa chamada é distinta das chamadas a mmThread::Start que iniciam as threads do renderizador.

Já CreateSurface, localizada em 0xd3a79c, chama vkCreateAndroidSurfaceKHR utilizando a janela retornada por mmCoreLibrary::GetWindow() (0xd3b2ec). Essa operação cria a superfície Vulkan associada à janela Android. Qualquer resultado diferente de zero é tratado como erro pelo wrapper analisado, provocando uma exceção.

Limitação: os parâmetros descritos identificam pontos que merecem investigação, não um defeito já comprovado. Ainda é necessário verificar as capacidades reais da superfície no aparelho e descobrir qual chamada Vulkan efetivamente retorna erro.

---------------------//----------------------

> Hipóteses e status

Esta tabela reúne as principais explicações consideradas até o momento. A distinção entre uma hipótese descartada, uma interpretação de alta confiança e uma causa confirmada é importante para não atribuir ao driver ou ao jogo um defeito que ainda não foi demonstrado. Ela será atualizada conforme novas descobertas foram sendo feitas, hipótese forem comprovadas, adicionadas ou descartadas, entre outros.

| Hipótese | Status | Evidência e observações |
|---|---|---|
| `pthread_create` falha e o handle fica inválido | **Descartada para o caminho analisado** | A criação das threads do renderizador ocorre no final de `Initialize`, depois das operações Vulkan que podem lançar a exceção. O fluxo de falha interrompe a inicialização antes da primeira chamada a `Start`. |
| Corrupção ou sobrescrita do `pthread_t` | **Improvável** | Os valores inválidos correspondem a sequências ASCII compatíveis com resíduos de heap. Isso favorece a hipótese de memória não inicializada, embora não descarte completamente outras formas de corrupção. |
| Handle de thread não inicializado | **Alta confiança** | A análise do construtor indica ausência de escrita em `this + 0x10`. O `Finalize()` tenta utilizar o handle antes de a thread correspondente ter sido iniciada. Falta confirmação em tempo de execução. |
| Exceção Vulkan antes da criação das threads | **Confirmada por análise estática** | O bloco `try/catch` cobre chamadas de inicialização Vulkan, e a ordem de execução coloca `mmThread::Start` depois dessas operações. Ainda falta identificar qual chamada lança a exceção. |
| Parâmetros do swapchain incompatíveis com o driver atual | **Hipótese aberta** | Merecem atenção `imageUsage = 19`, `compositeAlpha = INHERIT`, `preTransform = IDENTITY` e o uso das dimensões próprias em vez de `currentExtent`. As capacidades reais da superfície ainda não foram comparadas com esses valores. |
| Falha em `vkCreateAndroidSurfaceKHR` devido a uma janela inválida | **Hipótese aberta** | A função lança uma exceção diante de um resultado diferente de zero. Ainda não há evidência direta de que essa seja a chamada que falha. |
| Corrida com o diálogo de permissão | **Hipótese aberta, baixa prioridade** | O diálogo aparece aproximadamente 70 ms antes da inicialização gráfica. Essa proximidade temporal permite investigar uma possível condição de corrida, mas não demonstra relação causal. |
| Mudança de Android 14 para Android 16, One UI ou driver Vulkan | **Hipótese aberta** | O jogo funcionava no ambiente anterior e agora falha no atual. Sistema operacional, camada de interface e driver mudaram como parte do histórico, mas seus efeitos ainda não foram separados. Relatos de outros usuários sobre problemas semelhantes não foram verificados. |

---------------------//----------------------

Testes planejados

Os testes abaixo procuram separar as possíveis causas, começando por verificações simples e avançando para a instrumentação das chamadas Vulkan.

- [] Permissões: conceder manualmente as permissões do aplicativo antes de abri-lo e verificar se o crash persiste. O objetivo é testar a possível relação temporal com o diálogo de permissão.

- [] Configuração da tela: alterar a rotação, a resolução da tela e as opções do Game Booster, repetindo a inicialização após cada mudança. Isso pode ajudar a identificar uma relação com as dimensões da janela ou com o comportamento de apresentação.

- [] Capacidades Vulkan: utilizar um aplicativo de consulta às capacidades do dispositivo para obter supportedUsageFlags, supportedCompositeAlpha, supportedTransforms, currentTransform e currentExtent. Os valores devem ser comparados aos parâmetros observados em CreateSwapchain.

- [] Comparação entre aparelhos: consultar avaliações da Google Play Store em busca de relatos relacionados a diferentes plataformas, especialmente Exynos/Xclipse versus Snapdragon e Android 16 versus versões anteriores. Esses relatos devem ser tratados como indícios, não como evidência confirmada, até que sejam verificados.

- [] Ambiente alternativo: executar o jogo em outro aparelho ou emulador com uma versão anterior do Android, quando isso for possível, para verificar se o comportamento muda.

- [] Análise de OnAppCmd: desmontar e examinar melhor essa função, incluindo a atualização de s_deviceWidth e s_deviceHeight e a chamada a pthread_create().

- [] Análise de InitializeVulkanDevice: examinar a sequência de criação e inicialização do dispositivo Vulkan, buscando possíveis retornos de erro e exceções.

- [] Instrumentação Vulkan: se tecnicamente viável, instrumentar vkCreateInstance, vkCreateDevice, vkCreateAndroidSurfaceKHR e vkCreateSwapchainKHR para identificar qual operação retorna erro. Essa etapa é especialmente importante para passar de uma hipótese estática a uma causa demonstrada em execução.

Os testes devem ser realizados de forma controlada, alterando uma variável por vez sempre que possível. Assim, uma eventual mudança no resultado poderá ser associada com mais segurança ao fator testado.

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

A investigação, apoiada no método de Engenharia Reversa, segue, sempre que possível, uma abordagem incremental:

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
Assembly
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

🟢 Concluído / Confirmado

Reprodução e identificação do crash

- [x] Reproduzir o problema
- [x] Capturar o "logcat"
- [x] Identificar o "SIGABRT"
- [x] Identificar a mensagem relacionada a "pthread_join()"
- [x] Localizar a mensagem "invalid pthread_t ... passed to pthread_join"
- [x] Confirmar que o crash recente ocorre durante a inicialização do aplicativo
- [x] Rastrear o backtrace do crash recente até "mmRenderManager_Vulkan::Finalize()"
- [x] Relacionar o crash recente à primeira chamada a "WaitForEnd()" em "Finalize()", associada ao handle em "this + 0x3298"
- [x] Identificar que o valor inválido registrado no crash recente difere do valor observado em um crash anterior
- [x] Confirmar que a biblioteca analisada corresponde ao Build ID de um dos crashes registrados em 05/10/2026

Análise do gerenciamento de threads

- [x] Localizar "libultimateninjastorm.so"
- [x] Carregar a biblioteca no Ghidra
- [x] Localizar "mmThread_Impl::WaitForEnd()"
- [x] Identificar o acesso a "this + 0x10" em "WaitForEnd()"
- [x] Encontrar referências a "WaitForEnd()"
- [x] Localizar "pthread_create()"
- [x] Localizar "mmThread_Impl::Start()"
- [x] Identificar a chamada a "pthread_create()" dentro de "Start()"
- [x] Identificar o campo utilizado como identificador na chamada a "pthread_join()"
- [x] Identificar a chamada subsequente a "sched_setaffinity()"
- [x] Analisar o construtor "mmThread_ImplC2Ev()" e verificar que não apresenta uma escrita explícita em "this + 0x10" no trecho examinado
- [x] Identificar que não há verificação imediata visível do retorno de "pthread_create()" no trecho analisado de "Start()"
- [x] Localizar e analisar "mmThread_Impl::_Copy(...)"
- [x] Confirmar que "_Copy(...)" copia o campo "this + 0x10" do objeto de origem para o objeto de destino
- [x] Localizar e analisar "mmThread_Impl::SetAffinity(...)"
- [x] Confirmar que "SetAffinity()" utiliza os 32 bits inferiores de "this + 0x10" na chamada a "sched_setaffinity()"
- [x] Localizar o construtor de "mmThreadHandle"
- [x] Confirmar que o construtor aloca uma estrutura "mmThread_Impl" e armazena seu ponteiro no handle
- [x] Analisar o destrutor de "mmThreadHandle" e confirmar que ele destrói e libera a implementação associada
- [x] Analisar o destrutor de "mmThread_Impl" e verificar que ele não chama diretamente "pthread_join()" no trecho examinado
- [x] Identificar a ramificação de "mmThread_Impl::Start()" que configura o estado da thread por meio de "pthread_attr_setdetachstate()"
- [x] Analisar "mmRenderManager_Vulkan::Finalize()"
- [x] Identificar as duas chamadas a "mmThread::WaitForEnd()" em "Finalize()"
- [x] Identificar os handles associados às esperas em "this + 0x3298" e "this + 0x3320"
- [x] Investigar a possibilidade de o identificador ser não inicializado, obsoleto ou corrompido

Inicialização do renderizador Vulkan

- [x] Identificar o caminho de limpeza do renderizador Vulkan presente no backtrace recente
- [x] Analisar a sequência de inicialização do renderizador Vulkan e a criação de suas duas threads
- [x] Verificar a ordem entre as operações de inicialização Vulkan e as chamadas a "mmThread::Start()"
- [x] Identificar que as chamadas a "Start()" ocorrem depois das operações Vulkan examinadas na inicialização
- [x] Verificar que um caminho de falha anterior às chamadas a "Start()" pode conduzir à limpeza sem que essas threads tenham sido iniciadas por esse fluxo
- [x] Identificar o tratamento por "try/catch" no fluxo de inicialização Vulkan
- [x] Localizar a função "mmRenderManager_Vulkan::Initialize()"
- [x] Analisar os parâmetros de criação do swapchain identificados no código
- [x] Identificar o uso do primeiro formato retornado por "vkGetPhysicalDeviceSurfaceFormatsKHR"
- [x] Identificar o cálculo de "minImageCount" como mínimo solicitado mais uma imagem, limitado ao máximo quando aplicável
- [x] Identificar o uso de "mmCoreLibrary::s_deviceWidth" e "s_deviceHeight" para definir "imageExtent"
- [x] Identificar os valores analisados de "imageUsage", "preTransform", "compositeAlpha", "presentMode" e "clipped"

🟡 Em andamento

Ciclo de vida e validade do identificador da thread

- [ ] Rastrear o parâmetro que controla a ramificação de "pthread_attr_setdetachstate()" em "Start()"
- [ ] Investigar o tratamento dos retornos de "pthread_create()" e das funções de configuração dos atributos
- [ ] Verificar se o identificador em "this + 0x10" é inicializado ou alterado em outros pontos da biblioteca
- [ ] Investigar se o identificador armazenado em "this + 0x10" é utilizado corretamente em "pthread_join()" e "sched_setaffinity()"
- [ ] Investigar se copiar o campo "this + 0x10" entre objetos pode afetar o ciclo de vida das threads
- [ ] Compreender o ciclo de vida das threads: criação, configuração, espera e destruição
- [ ] Relacionar as funções de gerenciamento de threads à finalização do renderizador Vulkan
- [ ] Investigar a relação entre "pthread_create()", "sched_setaffinity()" e a utilização posterior do identificador
- [ ] Determinar por que o identificador passado a "pthread_join()" é considerado inválido
- [ ] Comparar os valores inválidos registrados nos diferentes crashes e verificar se revelam um padrão
- [ ] Verificar se os bytes ASCII presentes nos identificadores inválidos correspondem a resíduos de memória e em que circunstâncias aparecem
- [ ] Determinar se o campo não inicializado no construtor examinado é efetivamente a origem do identificador inválido
- [ ] Verificar se o estado da thread permite chamar "WaitForEnd()" antes de uma criação bem-sucedida

Falha de inicialização Vulkan

- [ ] Identificar qual operação ou exceção conduz o aplicativo ao caminho de limpeza do renderizador
- [ ] Rastrear as operações executadas imediatamente antes de "mmRenderManager_Vulkan::Finalize()"
- [ ] Identificar qual chamada Vulkan falha, caso uma exceção Vulkan seja a causa do caminho de limpeza
- [ ] Investigar a chamada a "vkCreateAndroidSurfaceKHR" e verificar se a janela utilizada é válida no momento da criação
- [ ] Comparar os parâmetros do swapchain com as capacidades reais da superfície Vulkan
- [ ] Verificar se "imageUsage = 19" é compatível com os usos suportados pela superfície
- [ ] Verificar se "compositeAlpha = INHERIT" está entre os modos de composição alfa suportados
- [ ] Verificar se "preTransform = IDENTITY" é compatível com as transformações suportadas
- [ ] Investigar o uso das dimensões próprias do jogo em vez de "currentExtent"
- [ ] Verificar se a escolha do primeiro formato de superfície pode produzir incompatibilidade
- [ ] Identificar se a falha ocorre durante a criação da superfície, a consulta das capacidades, a criação do swapchain ou outra etapa da inicialização

Comparação do ambiente e dos crashes

- [ ] Comparar os logs dos diferentes crashes, incluindo sequência temporal, mensagens e backtraces
- [ ] Determinar em quais condições cada chamada a "WaitForEnd()" falha
- [ ] Investigar se há relação temporal entre o diálogo de permissão e a inicialização gráfica
- [ ] Investigar possíveis diferenças de comportamento entre Android 14 e Android 16
- [ ] Avaliar possíveis efeitos da versão da One UI e do driver Vulkan
- [ ] Separar os efeitos do sistema operacional, da interface, do driver e do próprio aplicativo
- [ ] Verificar se há relatos independentes de falhas semelhantes em versões recentes do Android

🔴 Futuro

Comprovação da causa raiz

- [ ] Reunir as evidências estáticas e os logs em uma sequência causal única
- [ ] Determinar se o identificador inválido é a causa primária do crash ou uma consequência de uma falha anterior
- [ ] Confirmar a relação entre a falha de inicialização Vulkan, o caminho de limpeza e a chamada inválida a "pthread_join()"
- [ ] Identificar e documentar a causa raiz com evidências suficientes para reproduzir ou explicar o problema

Correção e validação

- [ ] Avaliar possíveis correções para o gerenciamento do ciclo de vida das threads
- [ ] Avaliar possíveis correções para a inicialização e a limpeza do renderizador Vulkan
- [ ] Desenvolver uma possível correção em ambiente de estudo controlado
- [ ] Testar a correção sem introduzir novos problemas de inicialização ou encerramento
- [ ] Validar o comportamento em diferentes tentativas de inicialização e encerramento
- [ ] Comparar os logs antes e depois da correção
- [ ] Documentar as limitações, os riscos e os resultados dos testes

Documentação e manutenção

- [ ] Documentar o processo completo de investigação, incluindo métodos, evidências e hipóteses descartadas
- [ ] Criar ferramentas auxiliares próprias para análise de logs, binários e backtraces
- [ ] Investigar outras incompatibilidades com versões modernas do Android
- [ ] Avaliar possibilidades de manutenção e compatibilidade do aplicativo
- [ ] Publicar os resultados e solicitar revisão técnica da comunidade, quando apropriado

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

> EXTRA: Como reproduzir o crash e capturar o log

O objetivo da captura é registrar o erro que acontece durante a inicialização sem expor informações pessoais ou dados de outros aplicativos.

1. Inicie a captura do logcat. Com o aparelho conectado ao computador por ADB, execute:

adb logcat -v threadtime

O logcat é o sistema de registros do Android. A opção -v threadtime acrescenta informações de data, hora, PID, TID e prioridade às mensagens, facilitando a reconstrução da sequência de eventos. Existem também formas de limitar as linhas do arquivo, uma vez que o esse tipo de captura costuma vir com uma quantidade exagerada de páginas, mesmo que ela dure apenas alguns poucos segundos. Entretanto, algumas linhas importantes podem se perder. Faça como achar melhor.

2. REPRODUZA O PROBLEMA. Abra o jogo e aguarde o encerramento. Volte ao aplicativo e pressione Ctrl+C para encerrar a captura. Procure a linha semelhante a Start proc ... com.bandainamcoent.ultimateninjastorm, que identifica o início do processo do aplicativo.

3. IDENTIFIQUE O PID DO JOGO. O PID é o identificador do processo. Ele permite distinguir as mensagens do jogo das mensagens produzidas por outros aplicativos e serviços do Android.

4. ISOLE AS MENSAGENS RELEVANTES. Preserve as linhas relacionadas ao PID do jogo, especialmente as mensagens de erro da biblioteca libc, o registro SIGABRT e o backtrace produzido pelo DEBUG. Como parte das informações relevantes pode ser emitida por serviços do sistema, registre também os trechos externos ao PID que sejam diretamente necessários para compreender a falha.

5. REVISE O ARQUIVO ANTES DE PUBLICÁ-LO. Remova informações pessoais e registros sem relação com o crash. Não publique o logcat completo: ele pode conter dados do aparelho, informações da operadora e mensagens de outros aplicativos, inclusive aplicativos bancários.

Um backtrace é a sequência de funções e endereços que mostra o caminho percorrido até a falha. Neste caso, ele é especialmente útil para relacionar o aborto registrado pelo Android à chamada a Finalize() e à tentativa de executar pthread_join() com um identificador inválido.

---------------------//----------------------

> Aviso legal

Este é um projeto independente, sem afiliação, associação ou endosso da Bandai Namco Entertainment, CyberConnect2 ou dos respectivos detentores de direitos.

Os direitos autorais referentes ao jogo, seu código, bibliotecas, personagens, imagens, músicas, marcas e demais conteúdos permanecem pertencendo aos seus respectivos detentores.

Este repositório não tem como objetivo redistribuir arquivos proprietários do jogo.

Arquivos como APKs, bibliotecas nativas, assets ou outros componentes proprietários não são disponibilizados neste repositório.

A análise utiliza nomes de funções, endereços, offsets e trechos curtos de evidência necessários para explicar os resultados, respeitando as limitações da investigação e os direitos dos titulares do software. Também está utilizando uma cópia obtida legalmente e arquivos extraídos localmente pelo pesquisador.

---------------------//----------------------

> Licença

Os textos, documentação, scripts e demais materiais originais criados especificamente para este projeto são disponibilizados sob a licença MIT, salvo indicação em contrário.

A licença MIT deste repositório não concede qualquer direito sobre o jogo NARUTO: Ultimate Ninja STORM ou sobre seus componentes proprietários.

---------------------//----------------------

Enfim, como podem ver, este projeto de estudo começou com um jogo que não inicia.

A intenção é terminar com uma compreensão muito maior de programação, sistemas, Android, engenharia reversa, debugging e desenvolvimento de software.

E, se no caminho for possível descobrir uma maneira de fazê-lo funcionar novamente, melhor ainda.

Muito obrigado pela atenção e peço paciência. Vou atualizando o repositório conforme os progressos forem surgindo.