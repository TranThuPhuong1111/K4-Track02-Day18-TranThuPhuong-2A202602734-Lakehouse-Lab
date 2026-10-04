# Thông tin bài nộp

| Mục | Giá trị |
|---|---|
| Họ tên | Trần Thu Phương (TranThuPhuong) |
| MSSV | 2A202602734 |
| Mã bài | K4-Track02-Day18 — Lakehouse Lab |
| Repo | https://github.com/TranThuPhuong1111/K4-Track02-Day18-TranThuPhuong-2A202602734-Lakehouse-Lab |
| Đường chạy | **Lightweight** cho cả 8 notebook (NB1–NB4 **không** dùng Spark) |
| Python | 3.11.9 (venv `.venv`) |
| Hệ điều hành | Windows 11 Home Single Language 10.0.26200, PowerShell |
| Ngày chạy | 2026-10-04 (UTC+7) |

## Phiên bản thư viện đã cài

```text
deltalake==1.6.6        pyiceberg==0.12.0      pyiceberg-core==0.10.1
duckdb==1.5.6           polars==1.44.2         pyarrow==24.0.0
numpy==2.4.6            SQLAlchemy==2.0.54     jupyterlab==4.6.4
jupytext==1.19.5        nbconvert==7.17.1      pytest==9.1.1
```

## Ghi chú môi trường (sự cố và cách xử lý)

Máy bật **Windows Smart App Control**, nên một số DLL trong wheel mới phát hành bị chặn
(`An Application Control policy has blocked this file`):

1. `pyarrow==25.0.1`: module `pyarrow._dataset` bị chặn nên smoke test lỗi ở bước Delta read.
   Đã cài `pyarrow==24.0.0`, vẫn nằm trong khoảng `pyarrow>=17,<26` của `requirements.txt`.
2. `SQLAlchemy==2.1.3` (dependency của `pyiceberg[sql-sqlite]`): `_processors_cy` bị chặn nên
   không tạo được `SqlCatalog`. Đã cài `SQLAlchemy==2.0.54`.

Không sửa mã nguồn notebook, assertion, ngưỡng hay `requirements.txt`; chỉ chọn phiên bản
dependency khác trong khoảng cho phép.

## Cách tạo notebook nộp

```powershell
Get-ChildItem notebooks/[0-9]*.py | ForEach-Object { .\.venv\Scripts\python.exe -m jupytext --to notebook $_.FullName }
.\.venv\Scripts\python.exe -m ipykernel install --sys-prefix --name lakehouse-lab
# thực thi từng notebook, giữ output
.\.venv\Scripts\python.exe -m nbconvert --to notebook --execute --inplace --ExecutePreprocessor.kernel_name=lakehouse-lab notebooks/<NB>.ipynb
Copy-Item notebooks/[0-9]*.ipynb submission/notebooks/
```

Số liệu trong [RESULTS.md](RESULTS.md) lấy từ output lưu trong `submission/notebooks/*.ipynb`.
Các log kiểm tra nằm trong [evidence/](evidence/).
