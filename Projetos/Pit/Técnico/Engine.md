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
 - PhysX adaptado (baixíssima propri)