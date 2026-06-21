# Pixel Art & Animação de Sprites

Como produzir arte pixel coesa e animada para um jogo. Cobre fundamentos,
resolução, paleta, animação, spritesheets, geração de personagens e o fluxo nas
ferramentas de edição.

---

## 1. Decisões que vêm ANTES de desenhar

1. **Resolução base do jogo.** Ex.: 320×180 (16:9), 256×224 (estilo SNES),
   160×144 (Game Boy). Tudo se mede em relação a ela.
2. **Tamanho de tile/sprite.** 8×8 (NES/GB), 16×16 (SNES/indie comum), 32×32
   (mais detalhe). Personagens costumam ocupar 1–2 tiles de largura.
3. **Densidade de pixel única.** NUNCA misture sprites de 16px com outros que
   "fingem" 32px na mesma cena — quebra a ilusão. Um pixel da arte = N pixels
   da tela, sempre o mesmo N.
4. **Paleta.** Defina 16–32 cores no total. Coesão de cor é o que separa "amador"
   de "profissional".

## 2. Pixel-perfect (regra técnica universal)

- **Filtro de textura = nearest** (nunca linear/bilinear). Em Godot:
  `Default Texture Filter = Nearest`. Em Phaser: `pixelArt: true`. Em Bevy:
  `ImagePlugin::default_nearest()`. Em Raylib/Pygame: render num buffer pequeno.
- **Renderize na resolução base e escale por inteiro** (×2, ×3, ×4). Escalas
  fracionárias (×2.5) causam pixels de tamanhos diferentes (shimmering).
- **Posições de câmera e sprite em inteiros** para evitar tremulação. Em
  movimento sub-pixel, arredonde a posição de renderização.
- Padrão baixo-nível: desenhar tudo numa `RenderTexture`/`Surface` do tamanho
  base e fazer um único *blit* escalado para a janela.

## 3. Fundamentos de desenho pixel

- **Silhueta primeiro.** Uma boa sprite é legível como sombra preta. Resolva a
  forma antes de cor/detalhe.
- **Contraste de luz, não só de matiz.** Garanta que valores (claro→escuro)
  leiam bem em escala de cinza.
- **Hue shifting:** ao escurecer, shifte o matiz para o azul/roxo; ao clarear,
  para amarelo/laranja. Sombras puramente "mais escuras da mesma cor" ficam
  mortas.
- **Anti-aliasing manual seletivo:** suavize curvas colocando 1 pixel de tom
  intermediário em cantos — com parcimônia, ou vira borrão.
- **Evite "jaggies"/banding:** numa linha diagonal, mantenha o comprimento dos
  segmentos consistente (ex.: 2,2,2 e não 1,3,2). Banding = dois degraus
  paralelos que formam uma escada feia.
- **Dithering:** alterne pixels de duas cores (padrão xadrez) para simular um
  tom intermediário ou gradiente com paleta limitada.
- **Outline:** contorno escuro (selective/black) destaca a sprite do fundo.
  Contorno interno ("self-outline" com cor da própria peça) é mais sutil.
- **Cluster/legibilidade:** trabalhe em grupos de pixels (clusters), evite
  "pixels órfãos" soltos que viram ruído.

## 4. Anatomia da animação 2D

Frames mínimos por estado (já vendem o movimento):

| Estado | Frames | Notas |
|---|---|---|
| Idle | 2–4 | respiração leve; nunca estático total |
| Walk/Run | 4–8 | ciclo: contato → baixo → passagem → alto |
| Jump | 3 | impulso → ápice → queda |
| Attack | 3–5 | antecipação → golpe → recuperação |
| Hit/Damage | 1–2 | recuo + flash branco |
| Death | 4–6 | maior peso dramático |

Princípios de animação aplicados a pixel:
- **Antecipação** (recuo antes do golpe) e **follow-through** (continuação após).
- **Squash & stretch:** comprime no impacto/agachamento, estica no salto.
- **Timing:** controle a velocidade pelo nº de frames por pose, não só pelo FPS.
- **Onion skin:** veja o frame anterior/seguinte semitransparente para manter
  consistência — recurso central das ferramentas abaixo.

## 5. Spritesheets

- Organize frames numa grade regular (ex.: 16×16 por célula, N colunas).
- Em código, recorte por índice: `frame = (col, row)`; a engine lê
  `frameWidth/frameHeight`.
- Mantenha **margens/padding consistentes** ou zero — padding irregular gera
  "bleeding" (pixels vazando entre frames ao escalar). Se houver bleeding,
  adicione *extrude* (replicar a borda) ou padding de 1–2px.
- Convenção comum: cada **linha = uma animação**, cada **coluna = um frame**.

## 6. Geração de personagens em camadas (estilo LPC)

Para muitos personagens/NPCs sem desenhar cada um, use **spritesheets em
camadas** padronizadas (o conjunto LPC — Liberated Pixel Cup — é o padrão
aberto):
- Um **esqueleto/base** com animações fixas (walk, slash, thrust, cast, shoot,
  hurt) em 4 direções, todas alinhadas ao mesmo grid.
- Camadas empilháveis: corpo → roupa → cabelo → armadura → arma. Cada camada é
  um PNG no mesmo layout; compô-las gera variações infinitas.
- Vantagem: troca de equipamento em runtime = trocar a textura da camada.
- Mantenha SEMPRE o mesmo número de frames e a mesma ordem entre camadas, senão
  desalinha.

## 7. Ferramentas de edição (e quando usar cada uma)

| Ferramenta | Tipo | Forte em | Use quando |
|---|---|---|---|
| **Pixelorama** | Desktop, open-source | Sprites, tilesets, animação, camadas, onion skin, exportação de spritesheet, paleta | Fluxo completo grátis, alternativa ao Aseprite |
| **LibreSprite** | Desktop, open-source | Onion skin, camadas, frames, timeline de animação (fork livre do Aseprite antigo) | Quer o fluxo "Aseprite clássico" sem custo |
| **Piskel** | Web/Desktop | Sprites e animação rápidos no navegador, export GIF/PNG | Protótipo rápido, sem instalar nada |

Recursos que você vai usar em qualquer uma: **camadas**, **timeline de frames**,
**onion skin**, **paleta indexada**, **espelhar/rotacionar**, **exportar
spritesheet** (PNG + dados de frames).

Fluxo típico:
1. Crie o documento na resolução do sprite (ex.: 16×16) com a paleta carregada.
2. Desenhe o frame idle base em camadas (silhueta → cor → sombra → luz).
3. Duplique frames e ajuste para criar o ciclo (use onion skin).
4. Exporte a spritesheet (grade regular) + anote frameWidth/frameHeight.
5. Importe na engine e configure as animações por linha/coluna.

## 8. Paletas e referências

- Comece de uma **paleta pronta e testada** (ex.: paletas de 16–32 cores
  populares como "PICO-8", "Endesga", "AAP-64") em vez de inventar do zero.
- Reuse cores entre sprites e tiles para coesão (a mesma rampa de sombra serve
  vários objetos).
- Mantenha um arquivo de **referência/estudo**: ciclos de caminhada, rampas de
  cor, exemplos de dithering e outline. Estudo dirigido > tentativa e erro.

## Checklist de arte pronta

- [ ] Resolução base e tamanho de tile definidos e respeitados em tudo.
- [ ] Paleta fechada (16–32 cores) e reutilizada entre sprites/tiles.
- [ ] Filtro nearest + escala inteira configurados na engine.
- [ ] Sprite com idle + walk + ação principal animados.
- [ ] Spritesheet exportada em grade regular, sem bleeding.
- [ ] Outline/contraste garantindo legibilidade sobre o fundo do jogo.
