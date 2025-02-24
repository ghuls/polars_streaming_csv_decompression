# Polars IO plugin for reading compressed CSV/TSV files in a streaming fashion

This plugin provides a way to read compressed CSV/TSV files in a streaming fashion for usage with [Polars](https://pola.rs/).

Currently, [Polars](https://pola.rs/) decompresses compressed CSV/TSV files completely in memory
(when using `pl.read_csv("file.csv.gz")` or `pl.scan_csv("file.csv.gz")`) before trying to parse them, which results in
a lot of memory usage when reading large compressed CSV/TSV files (several GBs to 100s of GBs) as common in e.g. bioinformatics.

This plugin provides a way to read compressed CSV/TSV files in a streaming fashion, where the file is decompressed and
parsed in chunks. This results in a much lower overall memory usage when reading large compressed CSV/TSV files.

As it is mainly intended for reading large compressed CSV/TSV files produced by bioinformatics tools, records are
assumed to be separated by `eol_char` (=`"\n"` by default) and embedded `eol_char` in fields are not expected. The last
record also should end in `eol_char`. If those conditions are not met, reading such files could give corrupt data.

It can also be used for decoding CSV files with a different character encoding than `utf8` and/or for decoding CSV files for
which not all bytes can be decoded in that encoding. Compared with `read_csv`, the decoding will require a lower amount of
total memory.

Streaming decompression is handled by [xopen](https://github.com/pycompression/xopen/), which supports the following compression
formats and backends and automatically selects the best backend available on the system:
  - gzip (`.gz`):
    - [python-isal](https://github.com/pycompression/python-isal)
    - [python-zlib-ng](https://github.com/pycompression/python-zlib-ng)
    - [pigz](https://zlib.net/pigz/) (a parallel version of gzip)
    - [gzip](https://www.gnu.org/software/gzip/)
  - bzip2 (`.bz2`):
    - [pbzip2](http://compression.great-site.net/pbzip2/) (parallel bzip2)
  - xz (`.xz`):
    - [xz](https://github.com/tukaani-project/xz)
  - Zstandard (`.zst`) (optional)":
    - [zstd](https://github.com/facebook/zstd)
    - [zstdandard](https://github.com/indygreg/python-zstandard): Install with `pip install xopen[zstd]`
  - fallback to Python’s built-in functions (`gzip.open`, `lzma.open`, `bz2.open`) if none of the other methods can be
    used.


## Installation

```bash
pip install git+https://github.com/ghuls/polars_streaming_csv_decompression.git
```

## Usage

```python
import polars as pl
import polars_streaming_csv_decompression

# Read compressed CSV file in a streaming fashion.
(
    polars_streaming_csv_decompression.streaming_csv(
        "my_big_file.csv.gz"
    )  # lazy, doesn't do a thing
    .select(
        ["a", "c"]
    )  # select only 2 columns (other columns will not be read)
    .filter(
        pl.col("a") > 10
    )  # the filter is pushed down the scan, so less data is read into memory
    .head(100)  # constrain number of returned results to 100
)


# Read CSV file with non-utf8 encoding in a streaming fashion.
(
    polars_streaming_csv_decompression.streaming_csv(
        "file_encoded_in_windows-1252.csv",
        encoding="windows-1252",
    )
    .head()
)

# Read CSV file with non-utf8 encoding where not all bytes can be decoded in a streaming fashion.
(
    polars_streaming_csv_decompression.streaming_csv(
        "file_encoded_in_windows-1252_but_not_all_bytes_can_be_decoded.csv",
        encoding="windows-1252-lossy",
    )
    .head()
)
```
