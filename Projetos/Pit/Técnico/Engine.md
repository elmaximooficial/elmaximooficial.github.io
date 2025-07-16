A engine será baseada em um ECS desmembrante, com alocação dinâmica limitada a um threshold de memória (>8KB transferências para o heap).

Cada Componente será armazenado com seus campos cada um em uma coluna.

O grafo de agendamento d sistemas deve ser feito em Comptime, com reordenacao possível em runtime.

Devemos ter relações de entidades, armazenados como ZSTs, codificados na assinatura de tipos do arquétipo.

Eventos serão modulados como interrupts do sistema.

O jogo deve rodar em: Windows, Linux, MacOS, PS4, PS5, XBOX, Android, iOS. Para tanto, vamos utilizar o winit com wgpu adaptado para ECS.

Os gráficos serão 3D poligonal, com iluminação Deferred e Transições entre Toon Shading normal e um shading mais "PS2"/Silent Hill no mundo invertido.

A câmera será top-down com ângulo de 45° em relação ao solo, com possíveis movimentações dinâmicas e programáticos da câmera para cutscenes e efeitos dramáticos.

O áudio deve ser 3D com efeitos comuns (echo, Doppler, decaimento, etc).

A física sera Rigid Soft Body, a engine ainda está para ser definida, porém temos:
 - Avian3D adaptada
 - PhysX adaptado (baixíssima prioridade devido a complexidade )
 - Jolt adaptado (baixa prioridade devido a ma integração com ECS)
 - Customizada (prioridade Alta devido ao potencial de performance e features.)

Devemos modelar FSMs de uma forma similar ao Sorcery, mantendo máquinas de estados para os encontros do jogador armazenados em arquétipos separados, relacionados ao jogador por EntityRelations (ex.: HasDone<Player, States<WolfEncounter>>).

Os controles deverão obrigatoriamente ser mapeados de forma dinâmica, permitindo ao jogador mudar qualquer configuração. No PC devemos fornecer controles adaptados ao QWERTY e ao DVORAK sem discussão (já sofri muito com isso).

O pipeline gráfico é o seguinte:
 - GraphicsSchedule é executado logo após PhysicsSchedule e antes de PreUpdate
 - os sistemas gráficos irão fazer um query a objetos renderizaveis (Mesh, Texture2D, TextureAtlas, Texture3D, etc)
 - esses elementos são handles para regions da GPU, deve ser checado de esses objetos sofreram alteração (através de um sinal de interrupt) e re-uploaded ou atualizados na GPU caso sim, caso contrário, mantenha-os iguais

Shaders utilizarão WGSL, padrão do wgpu, porém devemos permitir (através de bindings ou criando ferramentas de transpilacao) GLSL e HLSL. Shaders são comptime generated, devemos analisar o shaders e descobrir os parâmetros controláveis por CPU, a partir disso, gerar uma estrutura que contenha todos os parâmetros alteraveis via CPU.