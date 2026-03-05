# WeatherBench 2 ERA5 EDA 노트북 구축 프로세스

## 1. 목표

- WeatherBench 2 ERA5 Zarr 데이터(GCS 공개 버킷)를 xarray로 불러와 EDA 수행
- 서울 좌표(37.56°N, 126.97°E) 기준 2020년 데이터 필터링
- 월별 총 강수량을 바 차트로 시각화
- 결과물: `wb2_eda.ipynb`

---

## 2. 노트북 생성 (`wb2_eda.ipynb`)

**노트북 구성 (7개 섹션)**

| 섹션 | 내용 |
|------|------|
| 0. 패키지 임포트 | xarray, gcsfs, matplotlib, pandas, numpy |
| 1. 데이터 로드 | GCS 익명 접근 + `xr.open_zarr()` + Dask lazy loading |
| 2. EDA | 차원/좌표/변수 목록/시간 범위/격자 간격 확인 |
| 3. 필터링 | 서울 `method='nearest'` + 2020년 슬라이스 |
| 4. 월별 집계 | 6h 누적 강수량 합산 → mm 변환 |
| 5. 바 차트 | 월별 강수량 시각화 (강도별 색상, 기준선, 수치 레이블) |
| 6. 시계열 | 6h 강수량 + 7일 이동평균 플롯 |
| 7. 요약 통계 | 월별 강수량 테이블 + 비율 |

**핵심 구현**
```python
# GCS 익명 접근
fs = gcsfs.GCSFileSystem(token='anon')
ds = xr.open_zarr(store, consolidated=True, chunks={'time': 100})

# 서울 필터링
tp_seoul_2020 = ds['total_precipitation_6hr'] \
    .sel(time=slice('2020-01-01', '2020-12-31')) \
    .sel(latitude=37.56, longitude=126.97, method='nearest')
```

---

## 3. 환경 구축 (Miniconda)

### 3-1. Miniconda 설치
```bash
# 설치 스크립트 다운로드
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/Miniconda3-latest-Linux-x86_64.sh

# 자동 설치 (~/miniconda3)
bash ~/Miniconda3-latest-Linux-x86_64.sh -b -p ~/miniconda3

# 셸 초기화
~/miniconda3/bin/conda init bash
```

### 3-2. Anaconda ToS 동의 (신규 설치 시 필요)
```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### 3-3. 전용 환경 생성 및 패키지 설치
```bash
conda create -n ai4weather python=3.12 -y
conda activate ai4weather

pip install ipykernel xarray zarr gcsfs fsspec matplotlib pandas numpy dask
```

### 3-4. Jupyter 커널 등록
```bash
python -m ipykernel install --user --name=ai4weather --display-name "Python (ai4weather)"
```

> VS Code에서 커널 선택 시 **"Python (ai4weather)"** 선택

---

## 4. 트러블슈팅

### 문제 1: `ipykernel` 패키지 없음
```
Running cells with 'Python 3.12.3' requires the ipykernel package.
```
**원인:** 시스템 Python에 ipykernel 미설치
**해결:** conda 환경 생성 후 ipykernel 설치

---

### 문제 2: Dask 없음
```
ImportError: chunk manager 'dask' is not available.
```
**원인:** `xr.open_zarr(chunks=...)` 사용 시 dask 필요
**해결:**
```bash
pip install dask
```

---

### 문제 3: matplotlib 한글 깨짐
**원인:** Linux 기본 환경에 한글 폰트 미설치
**해결 순서:**

```bash
# 1. 한글 폰트 설치
sudo apt-get install -y fonts-nanum

# 2. matplotlib 캐시 삭제 및 갱신
python -c "import matplotlib as mpl, shutil; shutil.rmtree(mpl.get_cachedir())"
```

노트북 임포트 셀에 아래 코드 추가:
```python
import matplotlib.font_manager as fm

NANUM_PATH = '/usr/share/fonts/truetype/nanum/NanumGothic.ttf'
fm.fontManager.addfont(NANUM_PATH)
prop = fm.FontProperties(fname=NANUM_PATH)

plt.rcParams['font.family'] = prop.get_name()
plt.rcParams['axes.unicode_minus'] = False  # 마이너스 기호 깨짐 방지
```

---

## 5. VS Code 커널 재시작 방법

| 방법 | 실행 |
|------|------|
| 버튼 | 노트북 상단 툴바 **Restart (↺)** 클릭 |
| 단축키 | `Ctrl+Shift+P` → `Jupyter: Restart Kernel` |
| Run All | `Ctrl+Shift+P` → `Jupyter: Run All Cells` |

---

## 6. 최종 파일 구조

```
AI4WeatherandClimate/
└── wb2_eda.ipynb                           # 메인 노트북
└── seoul_monthly_precipitation_2020.png    # 월별 강수량 바 차트 (실행 후 생성)
└── seoul_timeseries_precipitation_2020.png # 시계열 플롯 (실행 후 생성)

~/miniconda3/envs/ai4weather/               # conda 환경
```
