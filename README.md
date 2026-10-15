# Roblox 3D 모델 저장소

Blender 스크립트로 로블록스용 3D 모델을 만들고 FBX로 내보내는 저장소입니다.

## 폴더 구조

| 폴더 | 내용 |
|------|------|
| `tools/` | 모델 생성·내보내기용 Blender Python 스크립트 |
| `exports/` | 로블록스 스튜디오로 가져올 FBX 파일 |
| `previews/` | 모델 미리보기 렌더 이미지(PNG) |

에셋 규칙은 [CLAUDE.md](CLAUDE.md)에 있습니다.

## 환경 설정 (Ubuntu)

```bash
apt-get install -y blender python3-numpy   # FBX 내보내기에 numpy가 필요함
```

Claude Code 클라우드 세션에서는 `.claude/hooks/session-start.sh`가 세션 시작 때 자동으로 설치합니다.

## 실행

```bash
blender --background --python tools/make_test_cube.py
```

`exports/Test_Cube.fbx`와 `previews/Test_Cube.png`가 생성됩니다.

- 실행하면 삼각형 수와 메시 검사 결과(n-gon, 떠 있는 점, 열린 모서리)를 출력합니다.
- 렌더는 GPU가 없어도 되도록 Cycles CPU를 씁니다. apt판 Blender에는 OpenImageDenoise가 없어서 디노이즈는 꺼져 있습니다(16샘플이라 약간 거칠게 나옵니다).

## 공룡 만들기

```bash
blender --background --python tools/make_dino.py -- TRex Trike   # 원하는 종만
blender --background --python tools/make_dino.py -- --all        # 전체
```

- 종별 값은 `tools/dinos.json`에 있습니다: 자세(`biped` 두발, `quad` 네발, `flyer` 날기, `swimmer` 바다), 키, 색, `params`(비율), `features`(특징). `based_on`으로 다른 종을 물려받을 수 있습니다(IceRex ← TRex 등).
- 특징은 `tools/make_dino.py`의 `FEATURES`에 있습니다. 각 특징은 비율 기본값(예: `croc_snout` → 긴 주둥이)과 덧붙일 모양(예: `back_sail` → 등 돛)을 가집니다. 새 종은 json에 한 줄 추가하고, 새 특징만 함수로 추가하면 됩니다.
- 결과: `exports/Dino_{ID}.fbx`, `previews/Dino_{ID}.png`, 여러 종이면 `previews/Dinos_All.png`(json 순서대로 이름표를 붙여 한 장에).
- 머리·턱·목·몸통·다리·팔·날개·지느러미·꼬리를 각각 따로 만들고 같은 이름의 버텍스 그룹을 붙인 뒤 한 메시로 합칩니다(나중에 뼈대 넣기용).
- 실행할 때마다 삼각형 수(3,000 이하), n-gon, 떠 있는 점, 열린 모서리, 최소 두께(0.1)를 검사하고, 어긴 종은 내보내지 않고 실패로 표시합니다. 두발 종의 머리 비율(볏·뿔 같은 장식 제외)이 35~40%를 벗어나면 요약에 표시합니다.
- 바다 종은 키 대신 몸 길이(코끝~꼬리 끝)로 크기를 맞춥니다(기본 5, `length`로 지정). 땅에서 0.5 떠 있습니다(기준점은 바닥 z=0).
- 크기를 줄이다가 작은 부분이 0.1보다 얇아지면, 그 부분만 두껍게 해서 다시 만듭니다.
- `previews/Dinos_All.png`는 몇 종만 다시 만들어도 사진이 있는 모든 종을 모아 새로 만듭니다.

### 색 (버텍스 컬러)

- 로블록스 3D 가져오기는 텍스처 없는 재질 색은 가져오지 않고 버텍스 컬러는 가져옵니다. 그래서 몸통·포인트·눈 색을 각 면의 버텍스 컬러(`Color`, sRGB)에도 넣어서 FBX로 내보냅니다. 재질은 블렌더와 미리보기 렌더용으로 그대로 둡니다.
- 스튜디오에서 가져올 때 "Ignore Vertex Colors"는 꺼 둡니다(기본값).

### 뼈대

- 모든 공룡에 뼈대(`Dino_{ID}_Rig`)가 들어가고, FBX에 메시와 함께 내보냅니다. 애니메이션은 없습니다.
- 공통 뼈: `Root`(기준점, 변형 안 함), `Hips`, `Spine`, `Neck`, `Head`, `Jaw`, `Tail1`~`Tail3`.
- 팔다리마다 뼈 3개: 다리 `UpperLeg_/LowerLeg_/Foot_`, 날개 `UpperWing_/LowerWing_/WingTip_`, 지느러미 `UpperFlipper_/LowerFlipper_/FlipperTip_`. 두발·날개는 `_L/_R`, 네발·지느러미는 `_FL/_FR/_BL/_BR`. 두발 15개, 나머지 21개.
- 뼈 위치는 몸을 만들 때 쓴 관절 위치를 그대로 씁니다. 가중치는 자동 가중치(bone heat)를 조각(닫힌 덩어리)마다 따로 계산하고 한 메시로 다시 합칩니다. 그래도 가중치가 없는 점은 가장 가까운 뼈에 붙입니다.
- 로블록스 제한에 맞춰 한 점에 뼈 4개까지만 영향을 주고, 가중치 합은 1로 맞춥니다.
- 확인용 사진 `previews/Dino_{ID}_rig.png`는 `Head`와 `Tail1`을 20도 돌려 렌더한 뒤 원래 자세로 되돌립니다. 모아 보기는 `previews/Dinos_All_rig.png`.

## 좀비 공룡 (밤의 적)

- `blender --background --python tools/make_dino.py -- Zombie`: `tools/dinos.json`의 "Zombie" 항목(두발, 통통한 민트빛 몸, 보라 얼룩 `zombie_patches`, 꿰맨 자국 `zombie_stitches`, 머리 반창고 `head_bandage`, 반쯤 감긴 눈꺼풀 `sleepy_lids`). 뽑기 공룡은 아니고 밤에 나오는 좀비 전용.
- 결과: `exports/Dino_Zombie.fbx`(삼각형 2,650개, 뼈 15개), `previews/Dino_Zombie.png`. 다른 공룡처럼 ServerStorage.Assets에 넣으면 좀비들이 이 모델을 씀.

## 적 몬스터 (화석 뼈 공룡)

```bash
blender --background --python tools/make_monster.py                       # 전부
blender --background --python tools/make_monster.py -- Monster_BonePtera  # 이것만
```

- `tools/make_monster.py`가 `Monster_BoneRaptor`, `Monster_BoneTrike`, `Monster_BoneStego`, `Monster_BonePtera`, `Monster_BoneRex`와 `Boss_Crown`을 만듭니다. 모양 만들기·뼈대·내보내기·렌더는 `make_dino.py`의 것을 그대로 씁니다.
- 통통한 만화풍 뼈: 척추 막대와 마디, 속이 빈 갈비뼈, 양끝이 둥근 다리뼈, 눈구멍이 파인 머리뼈. 눈구멍은 색 대신 머리뼈를 깊게 파서(불리언) 그림자로 어둡게 보입니다.
- 재질은 완전 흰색 하나뿐이고 버텍스 컬러도 없습니다. 게임에서 MeshPart 색을 레벨별로 입히면 됩니다.
- 키 2.5(BoneRex 3), 걷는 자세(BoneRaptor는 앞으로 기운 달리기 자세). BonePtera는 땅에서 1 떠 있고, 찢어진 막 조각이 손가락뼈 날개에 매달려 있습니다.
- 뼈대는 공룡과 같은 방식입니다(이름, 조각별 자동 가중치, 점마다 뼈 4개까지). 삼각형 2,000개 이하.
- `Boss_Crown`은 따로 된 오브젝트(뼈대 없음, 삼각형 400개 이하)로, BoneRex 머리뼈 폭에 맞춰 크기를 정합니다. 금색 재질(metallic 0.8)에 더해 금색 버텍스 컬러를 넣어 스튜디오에서도 금색으로 보입니다. 기준점은 왕관 바닥 가운데이고, BoneRex 모델 기준 머리뼈 꼭대기는 실행 요약에 출력됩니다.
- 사진: `previews/Monsters_All.png`(5종 + 왕관 + 왕관 쓴 BoneRex), `previews/Monsters_All_rig.png`(Head·Tail1을 돌린 확인용). 흰 뼈가 밝은 회색 배경에서 보이도록 모델에 닿는 주변광만 약하게 합니다(배경 밝기는 같음).

## NPC 티셔츠

```bash
blender --background --python tools/make_npc_shirts.py
```

- 가판대·게시판 NPC 공룡(이구아노돈·랩터·스테고·트리케라톱스)이 입는 상체 티셔츠 `NpcShirt_{ID}`를 만듭니다. `make_dino.py`로 같은 공룡을 다시 지어 몸통·배에 맞춘 몸판(밑단에서 자름), 짧은 소매(두발은 팔, 네발은 앞다리 윗부분), 가슴 동그라미 배지를 붙입니다.
- 공룡과 같은 기준점·크기로 내보내지만, 스튜디오로 가져오면 뼈대 있는 공룡과 옷의 기준점이 달라지므로 게임은 **상자 기준**으로 입힙니다: 같이 만들어지는 `NpcShirtFit.lua`의 크기·위치 차이를 써서, 공룡이 가져올 때 커진 만큼 옷도 키우고, 누워 들어왔으면 세우고, 공룡 머리 뼈로 앞을 찾아 제자리에 둡니다. 옷을 다시 만들면 이 파일도 바뀌니 Install.lua와 같이 커밋합니다. 색 2개(티셔츠, 소매 끝·배지), 뼈대 없음, 삼각형 600개 이하.
- 사진: `previews/NpcShirt_{ID}.png`(입은 모습), `previews/NpcShirts_All.png`.

## 효과음 (직접 합성)

```bash
blender --background --python tools/make_sounds.py                     # 42개 전부
blender --background --python tools/make_sounds.py -- SlapHit Thunder  # 이것만
```

- 녹음·다운로드 없이 계산으로 만듭니다(블렌더 파이썬의 numpy). 공기·비·흙은 걸러낸 잡음, 쇠·종·동전은 실제 막대·종의 배음 비율로 울리는 사인파, 공룡 목소리는 성대 같은 펄스를 입 모양(포먼트) 필터에 통과시키고, 작은 방 울림(리버브)을 섞습니다. 빗소리·폭풍은 끊김 없이 반복되는 10초 파일입니다.
- 결과: `exports/sounds/<이름>.wav`(모노 16비트 44.1kHz), 모양 확인용 `previews/Sounds_All.png`(파형과 스펙트로그램).
- **게임에 넣기**: 스튜디오 **Asset Manager → Bulk Import**로 wav 42개를 올리고, 각 오디오를 끌어다 놓아 생긴 Sound를 `SoundService` 안 `GameSounds` 폴더에 **파일 이름 그대로**(예: `SlapHit`) 넣으면 됩니다. 또는 올린 번호를 `Shared/Sfx.lua`의 `UPLOADED` 표에 적어도 됩니다. 없으면 로블록스 기본 소리가 대신 나고, 빗소리·폭풍 반복음은 올리기 전까지 소리가 없습니다.

### 배경음악

- 곡: `7out`, `mixymixy`, `ONE`, `uo`, `12-1-1`, `ffddfi` (mp3). `TWOw`는 뺐어요(GameMusic에 남아 있어도 안 틀어요).
- **게임에 넣기**: Asset Manager → Bulk Import로 mp3를 올리고, 생긴 Sound를 `SoundService` 안 `GameMusic` 폴더(없으면 Folder로 만듦)에 넣습니다 (그 폴더의 Sound는 이름과 상관없이 모두 곡). 플레이하면 출력 창에 `[Music] 배경음악 n곡` 줄이 뜹니다. 또는 올린 번호를 `StarterPlayerScripts/Music.client.lua`의 `UPLOADED` 표에 적습니다.
- 무작위 순서(한 바퀴 다 돌 때까지 반복 없음), 곡 사이 12초, 보기 설정 "배경음악"으로 끄고 켬.

## 미네랄과 열쇠

```bash
blender --background --python tools/make_props.py
```

- **미네랄** `Ore_Shard`(키 1.5), `Ore_Chunk`(3), `Ore_Boulder`(5), `Ore_Vein`(7, 폭 약 8): 울퉁불퉁한 회색 바위에서 통통한 육각 결정이 솟아 있습니다. 크기마다 FBX 하나에 메시 두 개가 들어 있습니다.
  - `Ore_{크기}_Crystal`: 재질 하나, 완전 흰색, 버텍스 컬러 없음. 게임에서 등급 색을 입힙니다.
  - `Ore_{크기}_Rock`: 회색(`#8C8F94`, 버텍스 컬러에도 넣음).
  - 두 메시 모두 기준점은 바위 아래 가운데(z=0)로 같습니다. `Ore_Vein`은 가운데 큰 결정 무리와 비탈을 도는 결정 고리로 여러 명이 동시에 캘 수 있게 했습니다.
- **열쇠** `Key_Normal`(은색 몸체 + 파란 띠), `Key_Gold`(금색 metallic 0.8 + 고리 위 빨간 보석): 길이 1.5, 세워진 모양, 둥근 고리에 세 발가락 공룡 발자국 구멍이 뚫려 있습니다. 기준점은 열쇠 가운데입니다. 색은 버텍스 컬러에도 넣었습니다.
- 사진: `previews/Props_All.png`. 열쇠 사진은 바닥에 묻히지 않도록 렌더할 때만 위로 올립니다(내보낸 FBX는 가운데 기준).

## 곡괭이

```bash
blender --background --python tools/make_pickaxe.py              # 15종 전부
blender --background --python tools/make_pickaxe.py -- 3 7 15    # 이 티어만
```

- 티어별 값은 `tools/pickaxes.json`에 있습니다: 머리 모양 키(`head`), 색(0 = 머리, 1 = 손잡이·장식, 2 = 추가 색), 손잡이·끝 뭉치 색, 손잡이 장식(`grip`: leather/bands/stars). 머리 모양 자체는 `make_pickaxe.py`의 `HEADS`에 있습니다.
- 모든 티어가 같은 손잡이를 씁니다: 길이 2.4(z = -0.5 ~ 1.9), 아래 끝 뭉치가 0.1 더 나옵니다. 크기를 다시 맞추지 않고 최종 크기로 바로 만들어서 손잡이 길이가 정확히 같습니다.
- 기준점은 손잡이 아래 끝에서 0.5 위(잡는 곳, z=0). 머리는 +Z, 곡괭이 날은 +Y/-Y. 머리는 티어가 한눈에 보이도록 1.2배 키워 그립니다(전체 높이 약 3~3.4).
- 색은 재질과 버텍스 컬러 둘 다에 넣습니다. 사진: `previews/Pickaxes_All.png`(곡괭이 사진은 바닥에 서도록 렌더할 때만 올림).

## 손바닥 장난감

```bash
blender --background --python tools/make_slap.py                  # 9종 전부
blender --background --python tools/make_slap.py -- Slap_Neon     # 이것만
```

- `Slap_Sponge`, `Slap_Rubber`, `Slap_Squeaky`, `Slap_BearPaw`, `Slap_GoldGod`(금색 metallic 0.8), `Slap_CatPaw`, `Slap_RoboArm`, `Slap_Neon`, `Slap_DragonClaw`: 짧은 막대 손잡이 위에 크게 펼친 손, 때리는 면은 +Y.
- 판정 크기가 같도록 9종 모두 손 부분(손가락·손바닥·손목)을 같은 상자에 딱 맞게 늘이거나 줄입니다: 폭 1.6(X) × 두께 0.6(Y) × 높이 1.8(Z), z = 0.9 ~ 2.7. 맞추다 작은 부분이 0.1보다 얇아지면 그 부분만 두껍게 해서 다시 만듭니다.
- 손잡이는 손 아래 1.2(z = -0.3 ~ 0.9, 끝 뭉치가 0.1 더 나옴). 기준점은 손잡이 아래 끝에서 0.3 위(잡는 곳, z=0).
- 색은 재질과 버텍스 컬러 둘 다에 넣습니다. BearPaw 발톱만 세 번째 색(어두운 갈색)을 씁니다.
- 사진: `previews/Slaps_All.png`. 때리는 면이 보이도록 렌더할 때만 180도 돌리고 바닥 위로 올립니다(내보낸 FBX는 그대로).

## 가방

```bash
blender --background --python tools/make_bag.py                   # 6종 전부
blender --background --python tools/make_bag.py -- Bag_Miner      # 이것만
```

- `Bag_Cloth`(키 1.6), `Bag_Leather`(1.8), `Bag_Miner`(2.0), `Bag_DinoEgg`(2.2), `Bag_RoboTank`(2.4), `Bag_Dimension`(2.6): 통통한 만화풍 배낭.
- 등에 닿는 평평한 면은 -Y(y=0), 그 면에서 어깨끈 두 개가 -Y 쪽으로 고리처럼 나옵니다(플레이어 어깨를 감쌈). 가방 바깥쪽은 +Y.
- 기준점은 등에 닿는 평평한 면의 가운데. 각 가방은 키에 맞춰 크기를 정하고, 줄이다 0.1보다 얇아진 부분은 두껍게 해서 다시 만듭니다.
- 색은 재질과 버텍스 컬러 둘 다에 넣습니다. 사진: `previews/Bags_All.png`(플레이어 뒤에서 보는 바깥쪽이 보이도록 렌더할 때만 180도 돌림).

## 맵 오브젝트

```bash
blender --background --python tools/make_map.py
```

- `Base_Set` 컬렉션(기지 한 세트, 게임에서 8개로 복제): `Base_FarmPen`, `Base_Shop` + `Shop_Sign`, `Base_DepositBin`, `Base_Arena`, `Base_SpawnPortal` + `SpawnPortal_Disc`.
- `Center_Set` 컬렉션(맵 가운데): `Zone_KeyAltar`, `Zone_Leaderboard` + `Leaderboard_Screen`, `Event_Meteor`.
- 모든 맵 오브젝트는 처음 요청 크기로 만든 뒤 1.5배(`MAP_SCALE`)로 키워 내보냅니다(캐릭터에 비해 작아 보여서). 아래 괄호 안이 실제 크기입니다.
- `Base_FarmPen`: 60 x 44(90 x 66) 나무 울타리(앞 -Y 쪽에 문), 잔디 바닥, 공룡 받침대 40개(8 x 5).
- `Base_Arena`: 바깥 반지름 26(39), 폭 6(9) 모래 트랙, 트랙 양쪽 낮은 경계석, 안쪽은 반지름 19(28.5) 단상(육각 타일 무늬, 받침대 없음). 바깥 경계석의 -Y 쪽 틈(반지름 39.5)에 소환문이 섭니다.
- 모두 따로 된 오브젝트이고, 각자 자기 이름의 하위 컬렉션에 들어 있습니다. 기준점은 바닥 가운데이고(평면 세 개는 평면 가운데), 회전·크기는 적용해 두고 위치만 세트 안의 자리로 남깁니다: 소환문은 아레나 바깥 경계석의 틈(-Y 쪽)에, 간판·화면·소용돌이 원판은 각자 가게·전광판·아치 위치에 붙어 있습니다.
- `Shop_Sign`, `Leaderboard_Screen`, `SpawnPortal_Disc`는 두께 없는 평면이라(SurfaceGui·효과용) 0.1 두께 규칙에서 뺍니다. 셋 다 -Y를 봅니다.
- 세트마다 FBX 하나: `exports/Base_Set.fbx`, `exports/Center_Set.fbx`. 사진: 오브젝트별 `previews/{이름}.png`, 세트 배치 `previews/Base_Set.png`·`Center_Set.png`, 모아 보기 `previews/Map_All.png`.

## 튜토리얼 에셋

```bash
blender --background --python tools/make_tutorial.py
```

- `Guide_Coco`: 막 태어난 아기 공룡 마스코트(키 2, 머리 52%). 민트 몸통, 크림 배, 어두운 눈(반짝이 두 개씩). 공룡과 같은 방식의 리그 `Guide_Coco_Rig`(뼈 21개)에 인사·가리키기용 팔 뼈 `UpperArm/LowerArm/Hand_L·R`이 들어 있습니다.
- `Coco_Shell`: 머리에 쓴 깨진 알껍데기 모자(흰색 + 하늘색 점). 따로 된 오브젝트지만 `Head` 뼈에 100% 붙어 머리를 따라 움직입니다. 셋 다 `exports/Guide_Coco.fbx` 하나로 나갑니다.
- `Coco_Egg` + `Coco_Egg_Top`: 키 2.5 알, 가운데 지그재그 금. 윗부분이 따로 된 오브젝트이고, 둘 다 기준점은 각자의 바닥 가운데입니다(윗부분은 높이 1.13에 놓임). `exports/Coco_Egg.fbx`. `previews/Coco_Egg_open.png`는 사진용으로만 윗부분을 들어 올린 모습입니다.
- `Tutorial_Dummy`: 나무 기둥에 꽂힌 공룡 인형 연습 허수아비(키 3.9). 황갈색 천, 빨간 과녁, 나무색 기둥. 꿰맨 자국과 배의 둥근 과녁. 기준점은 기둥 바닥.
- 사진: `previews/Guide_Coco.png`, `Guide_Coco_rig.png`(손 흔드는 포즈), `Coco_Egg.png`, `Coco_Egg_open.png`, `Tutorial_Dummy.png`, 모아 보기 `Tutorial_All.png`.

## 던지는 아이템

```bash
blender --background --python tools/make_items.py
```

- `Item_EggBomb`: 폭탄으로 쓰는 공룡알(심지·불꽃 포함 키 1.23, 알 몸통 0.84). 연두색 바탕에 진한 초록 점, 위에 짧은 갈색 심지와 주황색 불꽃.
- 기준점은 알 몸통의 가운데(바닥이 아님)라서 던졌을 때 가운데를 축으로 돕니다. 사진은 땅에 세워 보이도록 렌더할 때만 올립니다.
- 색은 요청대로 4가지(연두·진초록·갈색·주황)로, 규칙의 "최대 3색" 예외입니다. `exports/Item_EggBomb.fbx`, `previews/Item_EggBomb.png`.

## 맵 조립 (로블록스 스튜디오)

`roblox/MapBuilder.lua` 전체를 스튜디오 명령 모음(Command Bar)에 붙여넣고 한 번 실행하면 `Workspace.Map`에 맵 전체가 만들어집니다.

바닥이 레고 블록(초록 플라스틱 + 돌기)으로 안 보이면 `roblox/LegoFloor.lua`를 같은 방법으로 붙여넣어 실행하세요. 맵을 다시 짓지 않고 템플릿 회색 바닥을 지우고, 레고 판을 깔거나 높이를 맞추고, 바닥 파트를 돌기 플라스틱으로 바꾼 뒤 출력 창에 무엇을 찾고 고쳤는지 알려 줍니다. 다시 실행해도 기존 `Map` 폴더와 맵 영역 터레인(-500~500)을 지우고 새로 만듭니다.

- 블렌더 모델은 `ServerStorage.Assets` 안에서 오브젝트 이름으로 찾습니다(하위 폴더·모델 안까지). `exports/Base_Set.fbx`, `exports/Center_Set.fbx`를 3D 가져오기로 불러와 `Assets`에 넣으면 됩니다. 이때 메시를 하나로 합치지 않아야 이름이 남습니다. 없는 모델은 건너뛰고 출력창에 목록을 보여 줍니다.
- 모델은 바닥 가운데를 땅에 맞추고, 정면(-Z)이 정해진 쪽을 보게 돌려 놓습니다. 가져온 모델이 엉뚱한 쪽을 보면 스크립트 위쪽의 `ASSET_FRONT_YAW`(도)를 바꿉니다.
- 폴더 구성:
  - `Ground`: 광장, 둘레길, 흙 링, 주황 띠, 돌길.
  - `KeyZone`.
  - `OreSpawns`: 투명 파트 750개. 구역 구분 없이 고르게(서로 12 이상 떨어지게) 뿌립니다.
  - `Battlefield`: 투명·충돌 없는 원통, 반지름 360·높이 60.
  - `FieldDecor`: 광물 링(배틀필드) 위의 바위(대부분)와 나무 260개. 길과 광물 지점을 피합니다.
  - `BattlefieldFlags`.
  - `Base1`~`Base8`: 반지름 690, 폭 210 x 깊이 165 부지. 모델, 팀 색 `SpawnLocation`, 필드 표지판 `DefenseSign`·`FarmSign`, `Fence`, `SafeZone`.
  - `Decor`.
  - `Boundary`: 보이지 않는 벽과 언덕 위 나무.
- 부지 앞쪽에는 저금통(가운데)·가게(왼쪽)·스폰(오른쪽)이 있습니다. 깊이 60부터 뒤쪽은 낮은 울타리로 반씩 나눠 오른쪽이 디펜스 필드(아레나 + 소환문), 왼쪽이 수집 필드(농장)입니다. 필드 입구마다 표지판 파트가 있고, 표지판의 앞면(Front)이 입구 쪽을 봅니다.
- 맵은 1.5배 배치를 다시 2배로 넓힌 크기입니다(부지와 광장 크기는 그대로). 잔디 터레인은 반지름 960까지이고, 그 바깥은 언덕과 나무입니다. 반지름 960에 보이지 않는 벽이 있습니다. 나무·바위는 캐릭터의 4~7배 높이입니다.
- 기지 안에서 "앞"은 입구 쪽이고, 왼쪽·오른쪽은 입구로 걸어 들어가는 플레이어 기준입니다. 기지 1은 북쪽(-Z)에서 시계 방향 22.5도에 있고 45도 간격으로 이어집니다. 리더보드 뒤에 길이 나지 않게 하려고 이렇게 정했습니다.
- `SpawnLocation`은 `Neutral = false`, 팀 색 8가지(Bright red, Bright blue, Lime green, New Yeller, Bright violet, Deep orange, Toothpaste, Hot pink)입니다. 같은 색의 팀은 게임에서 만들어야 합니다.

배치도 `previews/map_layout.png`:

```bash
blender --background --python tools/draw_map_layout.py
```

이 스크립트는 `exports/Base_Set.fbx`·`Center_Set.fbx`를 불러와 모델 크기를 재고, 로블록스 API 흉내(`tools/roblox_mock.lua`)와 `luau` 명령으로 MapBuilder.lua를 실제로 두 번 연달아 돌리고, 만든 파트를 검사한 뒤 위에서 본 그림으로 그립니다. 그래서 배치도가 스크립트와 어긋나지 않습니다. 검사하는 것:
- 맵 폴더가 하나만 남는지.
- 모든 파트가 Anchored인지.
- 광물 지점이 750개이고 링 안에 고르게 퍼졌는지(구역 속성 없음).
- 모델·스폰·표지판이 부지의 정해진 구역 안에 있고, 방향과 바닥 높이가 맞는지.
- 두 필드 사이 울타리가 있고, 입구 폭 10이 비어 있는지. 부지 모서리가 벽 안쪽인지.
- 소환문이 트랙 틈에 있는지.
- Assets가 없을 때도 맵이 만들어지는지.

`luau`는 세션 시작 훅이 설치합니다.

## 게임 코드

기획서: [docs/design.md](docs/design.md). 모든 작업의 기준입니다.

```
roblox/src/ServerScriptService/    서버 스크립트 (이름.server.lua)
roblox/src/ReplicatedStorage/      공용 모듈. Config.lua = 기획서의 모든 수치
roblox/src/StarterPlayerScripts/   클라이언트 스크립트 (이름.client.lua)
roblox/src/StarterGui/             화면
```

설치 파일 만들기와 검사:

```bash
python3 tools/build_installer.py
```

- 결과 `roblox/Install.lua`를 스튜디오 명령 모음에 붙여넣고 한 번 실행합니다. 이전 설치분을 지우고 모든 스크립트를 제자리에 새로 만듭니다.
- 검사는 세 가지입니다: 모든 스크립트의 문법, Config(`tools/config_checks.lua`: 확률 합 100%, 표 길이, 기획서에 적힌 결과값), 흉내 환경(`tools/roblox_mock.lua`)에서 Install.lua를 두 번 실행해 소스와 똑같이 설치되는지.
- 코드 규칙은 CLAUDE.md의 "로블록스 게임 코드 규칙"에 있습니다.

서버 구성 (1단계 "기반"):

| 파일 | 하는 일 |
|---|---|
| `ServerScriptService/Main.server.lua` | 서비스들을 순서대로 시작 |
| `Services/PlayerData.lua` | 데이터 저장. 세션 잠금, 자동 저장, 나갈 때·서버 종료 때 저장, leaderstats(돈·열쇠·레벨) 표시 |
| `Services/ProfileLogic.lua` | 저장 규칙만 모은 순수 모듈 (잠금 판정, 기본값 채우기, 옛 저장 옮기기). `tools/tests/profile_logic_test.lua`로 테스트 |
| `Services/Policy.lua` | 접속 때 PolicyService로 유료 랜덤 아이템 제한 여부 확인 |
| `Services/MoveGuard.lua` | 걷기 속도를 서버가 정하고, 그보다 빨리 움직이면 되돌림 |
| `Services/TestCommands.lua` | 스튜디오 전용 채팅 명령 (`;help`) |
| `Services/Bases.lua` | 들어온 순서대로 기지 1~8 배정, 팀 색으로 자기 기지에서 부활 (2단계) |
| `Services/Minerals.lua` | 미네랄 190개 생성·재생성(크기 제각각), 탭하면 로딩 후 채취를 반복, 손에 든 곡괭이, 가방 상태 ("가방 가득!") |
| `StarterPlayerScripts/MiningFx.client.lua` | 캐는 사람 모두의 온몸 곡괭이질 (R15 관절을 직접 움직임, `tools/preview_mining_pose.py`로 확인), 내 로딩 막대 |
| `Services/Deposit.lua` | 자기 기지 입금통에서 가방을 돈으로 (2단계) |
| `Services/Shop.lua` | 자기 기지 상점(곡괭이·가방·캐릭터 강화), 옆 열쇠 가판대와 열쇠 판매, 스폰 발판 옆 🔄 환생 표지판과 환생 |
| `ReplicatedStorage/Shared/Formulas.lua` | 계산식 모음 (파워·속도·채취량·가방·입금액·뽑기). `tools/tests/formulas_test.lua`로 테스트 |
| `StarterPlayerScripts/Hud.client.lua` | 가방 막대, 곡괭이 정보, 상점 창 (2단계, 회색 박스 UI) |
| `Services/Defense.lua` | 주인이 접속한 기지마다 디펜스 자동사냥, 공룡 배치 요청 확인, 처치 보상, 단상 위 공룡 모델 (3단계), 승급전과 레벨 해금 (5단계) |
| `Services/DefenseLogic.lua` | 디펜스 판정만 모은 순수 모듈 (생성·이동·공격 4종·얼음/불/전기). `tools/tests/defense_logic_test.lua`로 테스트 |
| `StarterPlayerScripts/DefenseView.client.lua` | 서버가 보낸 위치·속도로 몬스터를 그리고 공격 효과 표시 (3단계) |
| `Services/Gacha.lua` | 뽑기: 열쇠 확인, 종·레어도·크기·변이 추첨, 공룡 추가, 시크릿 서버 공지 (4단계). 확률 계산은 `Formulas.gachaOdds`, `tools/tests/gacha_test.lua`로 테스트 |
| `StarterPlayerScripts/Gacha.client.lua` | 뽑기 창 (1번·10번, 결과 공개 연출)과 확률 보기 창 (4단계, 회색 박스 UI) |
| `Services/Settings.lua` / `StarterPlayerScripts/ViewSettings.client.lua` | 보기 설정 (내·남의 공룡·화석 숨기기), 저장됨 |
| `Services/Keys.lua` | 중앙 열쇠 생성·운석 샤워, 줍기·운반(느려짐)·기지 확정, 찰싹 맞으면 흩뿌리기 (6단계) |
| `Services/Combat.lua` | 손바닥 판정(사거리·정면·쿨다운·배틀필드), 넉백·그로기(체력 없음), 기지에서 다시 시작·무적, 현상금 1등 (6단계). 계산식은 `tools/tests/combat_test.lua`로 테스트 |
| `StarterPlayerScripts/Compass.client.lua` | 최상단 가운데 작은 원형 나침반: 내 기지 쪽 화살표와 거리(m) |
| `StarterPlayerScripts/CombatFx.client.lua` | 찰싹 버튼(F), 손바닥 휘두르기·"찰싹!" 연출, 넉백·그로기 카운트다운, 배틀필드 알림, 현상금 왕관 |
| `Services/Farm.lua` | 농장 등록·해제, 농장 우리의 공룡, 도감 기록(옛 저장은 보유 공룡으로 채움), 테마 완성 스킨, 보너스 속성, 도감 창 데이터 (7단계). 계산식은 `tools/tests/farm_test.lua`로 테스트 |
| `ReplicatedStorage/Shared/DinoVoices.lua` | 종마다 목소리(포효·울음·나팔·쉭쉭 짹짹·그르렁·끼익·바다 울음 7가지)와 음높이, 클수록 낮게 |
| `StarterPlayerScripts/Footsteps.client.lua` | 걷는 캐릭터마다 "자박자박" 발소리(걸음 간격은 걸은 거리로) |
| `StarterPlayerScripts/Npcs.client.lua` | NPC 공룡 숨쉬기·가끔 으르렁, 누를 때 동작·말풍선, 중앙 게시판 화면과 서버 순위 창 |
| `StarterPlayerScripts/Collection.client.lua` | 위쪽 줄 "도감" 창: 도감 수, 다음 보상, 테마별 수집, 지금 받는 보너스 |
| `ReplicatedStorage/Shared/Windows.lua` | 열린 창을 Esc·Backspace·게임패드 B로 닫기 (마지막에 연 창부터) |
| `Services/Enhance.lua` | 공룡 강화: 디펜스 아레나 앞 강화 가판대(상점 가판대 복제), 같은 종 공룡을 재료로 경험치·레벨업. 계산식은 `tools/tests/enhance_test.lua`로 테스트 |
| `Services/Npcs.lua` | 상점·열쇠·강화 가판대 안과 중앙 게시판 옆 NPC 공룡(티셔츠를 상자 기준으로 맞춰 입힘), 게시판 "게시판 보기" 버튼 |
| `Services/NpcShirtFit.lua` | `tools/make_npc_shirts.py`가 만드는 표: 공룡·티셔츠 상자 크기와 위치 차이 (손으로 고치지 않음) |
| `Services/OwnerSigns.lua` | 기지 입구 옆 주인 표지판: 아바타 얼굴, 닉네임, @아이디, 디펜스 LV·환생 |
| `Services/SpeedPads.lua` | 기지 밖 무작위 속도 발판: 밟으면 잠깐 1.5~2배 빨라짐 (중복 안 됨, MoveGuard.boost) |
| `Services/Weather.lua` | 날씨: 드물게 약 1분 비·천둥번개, 배너 알림, 그동안 디펜스 보상 2배 |
| `StarterPlayerScripts/WeatherFx.client.lua` | 하늘: 흘러가는 구름, 날아다니는 익룡, 비·번개·흐린 하늘, 나침반 아래 날씨 줄 |
| `Services/HomeTeleport.lua` | "기지로" 버튼: 3초 뒤 자기 기지 SpawnLocation으로 (열쇠를 들고 있으면 불가) |
| `ReplicatedStorage/Shared/DinoMotion.lua` | 공룡·뼈 몬스터의 뼈대를 움직여 걷기·숨쉬기·꼬리·날개·공격 모션 (화면 전용). `tools/preview_motion.py`가 같은 식으로 `previews/Motion_Check.png`를 그림 |
| `StarterPlayerScripts/Placement.client.lua` | 디펜스 현황 줄, 공룡 목록, 사거리 원을 보며 탭으로 배치, 배치 공룡 옆 이동·빼기 버튼, 강화 창 |
| `ReplicatedStorage/Shared/Net.lua` | 모든 통신. 서버가 형식·빈도·데이터 준비 여부를 확인 |
| `StarterPlayerScripts/Client.client.lua` | 알림 표시 |

경제 수치(곡괭이 가격, 파워·속도 강화 비용, 가방 비용)는 `python3 tools/economy_sim.py`로 계산합니다. 정해진 구매 순서대로 다음 구매까지 기다릴 시간(플레이 시간의 10%, 최대 약 1시간 43분)을 먼저 정하고(300시간 계획), 그동안 채굴로 버는 돈의 `PRICE_MULTIPLIER`(1.5)배를 가격으로 삼습니다. 그래서 기본 콘텐츠 전체는 약 450시간이 됩니다. 출력된 표를 Config.lua에 옮깁니다.

`python3 tools/income_sim.py`는 열쇠 줍기·광물 캐기·디펜스의 시간당 벌이를 페이스 표의 시점마다 비교해, 순서가 열쇠 > 광물 > 디펜스인지 확인합니다 (Config.lua 값을 읽음).

스튜디오에서 저장을 시험하려면 플레이스를 Roblox에 한 번 저장하고, 게임 설정 → 보안 → "API 서비스에 대한 Studio 액세스 사용"을 켜야 합니다. 꺼져 있으면 저장 없이 플레이되고 화면에 알려 줍니다.
