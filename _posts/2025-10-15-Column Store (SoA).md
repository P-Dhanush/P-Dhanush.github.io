---
layout: single
title: "Column Store (SoA)"
date: 2025-10-15
categories: [notes]      # or a single string: category: notes
tags: [jekyll, howto]
---

---

# Crypto L2 Analysis: Part 1 - Column Store Architecture

* TOC
{:toc}

---

## Workflow

Taking OKX L2 Data for BTC - USDT in an NDJSON format, we convert the file types for faster parsing. Since this is market by price and not market by order, we have some limitations with what we can do with the data.

**You can access the data [here](https://www.okx.com/en-us/historical-data)**
Take Order Book, spot data of depth 400. Any one day's data would suffice for exploration.

To begin with we have two types of actions:
- snapshot
`{"instId":"BTC-USDT","action":"snapshot","ts":"1758326400005","asks":[["115630.1","0.0145523","1"],["115631.0","0.0539","1"]...],"bids:[[px,qty,cnt],...[px,qty,cnt]]}`
- update
`{"instId":"BTC-USDT","action":"update","ts":"1758326400015","asks":[["115717.8","0.18242079","6"]],"bids":[]}`

For this stage of our implementation we focussing on converting the data into this structure:-
```cpp

    // A minimal, append-only columnar store with separate binary files per column.
    // Layout:
    //   outdir/
    //     schema.json              (human readable schema)
    //     columns/
    //       frame.u64
    //       ts_ms.u64                 (uint64 little-endian, ms since epoch)
    //       side.u8                (0 = bid, 1 = ask)
    //       etype.u8               (0 = SNAP, 1 = add, 2 = modify,3 = del, 4 = clear_bids, 5 = clear_asks)
    //       level.u16              (0..N-1 within that frame)
    //       px.f64                 (double price)
    //       qty.f64                (double size, amt. actually held)
    //       count.u32              (count, number of orders)
    //       inst_id.txt            (one line: instrument name)
    //     footer.json              ({"rows": <N>})
    // All files are append-only; caller is responsible for writing a consistent number of rows across columns.
```


### Directory Structure
```css
include -
|
|__mercury // (I named my project mercury)
| |
| |__colstore.hpp
| |__ndjson_okx.hpp
|
|__src
| |__colstore_inspect.cpp
| |__okx_to_colstore.cpp
```

### Strategy

#### Obtaining the parameters involved. (Read Stage)
```css
        ----------------------
       |                      | 
    ---|       PARSER         |
    |  |----------------------|
    |
    |  |- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function] 
    |--|- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function]
    |  |- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function]
    | 
    |  |- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function]
    |--|- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function]
       |- (px,qty,cnt) tuple -> [Obtain Parameters via Parser function]
```

**To keep in mind the possibility of a dynamic use-case of the parameters we obtain, we dont hardcode what we do with the obtained parameters in the PARSER function itself. Instead we send a `lambda` function in the PARSER function.**

```cpp
// The parse function structure in the header file ->
template <class F>
inline void parse_okx_ndjson(const std::string &path, F &&on_record)
{...}

// The parse function we call ->
mercury::parse_okx_ndjson(in_path, 
        [&](uint64_t ts_ms, uint8_t side, uint16_t level, double px, double qty, uint32_t count)
        {
            // Skip empty quantities (deletes will appear as qty=0 in some datasets)
            if (qty <= 0.0)
                return;
            writer.append(frame, ts_ms, side, level, px, qty, count);
            ++emitted;
            if ((emitted % 1'000'000) == 0)
            {
                cerr << "… " << emitted << " rows written\n";
            } 
        }
        );

        writer.close();
        cout << "Wrote " << writer.rows() << " rows to " << outdir << "\n";
```

#### Writing out the parameters per our structure.

The `writer` function seen above is to write the parameters we obtain down into our files.

- We configure a path where we can create our output columns.
- Create a ColWriter class with an apt constructor to open files on initilaizing. This can bring down overhead.
- We then use functions to write the parameters we've extracted into these files. The `lambda` funciton earlier takes care of this. We simply provide an apt `append` funciton to aid this process.

```cpp
struct Files{
  std::FILE *frame = nullptr;
  std::FILE *ts_ms = nullptr;
  std::FILE *side = nullptr;
  std::FILE *level = nullptr;
  std::FILE *px = nullptr;
  std::FILE *qty = nullptr;
  std::FILE *count = nullptr;
  fs::path dir;
  uint64_t rows = 0;
}
class ColWriter{
  public:
    // The constructor function will open up all the files.
    explicit ColWriter(filestream::path outdir, ) -> outdir_(sd::move(outdir)){
      check_directory_present(outdir); // if not the func will take care of creating.
      auto cols = outdir_ / "columns";
      check_directory_present(cols);

      files_.dir = outdir_;
      files_.frame = open_or_throw(cols / "frame.u64");
      files_.ts_ms = open_or_throw(cols / "ts_ms.u64");
      files_.side = open_or_throw(cols / "side.u8");
      // You get the idea.. (open all similarly, leaving for brevity)  
    }

    ~ColWriter{
      close();
    }

    ColWriter(const ColWriter &) = delete;
    ColWriter &operator=(const ColWriter &) = delete;

    void append(uint64_t frame, uint64_t ts_ms, uint8_t side, uint16_t level, double px, double qty, uint32_t count){
      write_scalar(files_.frame, frame);
      write_scalar(files_.ts_ms, ts_ms);
      // so on..
    }

    void safe_close(std::FILE*& f){ // A reference to the pointer. We want to set caller's pointer varaible to nullptr, so we can't let the pointer be passed by value and need actual pointer.

        if(f){
          std::fclose(f);
          f = nullptr;
        }
    }

    void close(){
      if(closed_){
        return;
      }

      safe_close(files_.frame);
      safe_close(files_.ts_ms);
      //so on..
    }



  private:
    filestream::path outdir_;
    Files files_{};

}
```


## The actual code used in each file:

### colstore.hpp
```cpp
#ifndef MERCURY_COLSTORE_HPP
#define MERCURY_COLSTORE_HPP

#include <cstdint>
#include <cstdio>
#include <cstring>
#include <string>
#include <string_view>
#include <vector>
#include <filesystem>
namespace fs = std::filesystem;

#include <stdexcept>
#include <memory>
#include <system_error>
#include <optional>
#include <fstream>

namespace mercury
{
    struct ColFiles
    {
        std::FILE *frame = nullptr;
        std::FILE *ts_ms = nullptr;
        std::FILE *side = nullptr;
        std::FILE *level = nullptr;
        std::FILE *px = nullptr;
        std::FILE *qty = nullptr;
        std::FILE *count = nullptr;
        fs::path dir;
        uint64_t rows = 0;
    };

    inline void ensure_dir(const fs::path &p)
    {
        std::error_code ec;
        if (!fs::exists(p) && !fs::create_directories(p, ec))
        {
            throw std::runtime_error("Failed to create directory: " + p.string() + ", error: " + ec.message());
        }
    }

    inline std::FILE *open_or_throw(const fs::path &p)
    {
        auto f = std::fopen(p.string().c_str(), "wb");
        if (!f)
            throw std::runtime_error("Failed to open for write: " + p.string());
        return f;
    }

    class ColWriter
    {
    public:
        explicit ColWriter(fs::path outdir, std::string_view inst_id)
            : outdir_(std::move(outdir))
        {
            if (inst_id.empty())
                throw std::invalid_argument("inst_id is empty");
            // dirs
            ensure_dir(outdir_);
            auto cols = outdir_ / "columns";
            ensure_dir(cols);

            // schema
            {
                std::ofstream s(outdir_ / "schema.json", std::ios::binary);
                s << "{"
                     "\"version\": 1,"
                     "\"columns\": ["
                     "{\"name\": \"frame\",   \"type\": \"u64\", \"unit\": \"integer - a count\"},"
                     "{\"name\": \"ts_ms\",   \"type\": \"u64\", \"unit\": \"ns\"},"
                     "{\"name\": \"side\", \"type\": \"u8\",  \"desc\": \"0=bid,1=ask\"},"
                     "{\"name\": \"level\",\"type\": \"u16\"},"
                     "{\"name\": \"px\",   \"type\": \"f64\"},"
                     "{\"name\": \"qty\",  \"type\": \"f64\"},"
                     "{\"name\": \"count\",  \"type\": \"u32\", \"desc\": \"number of orders\"}"
                     "]"
                     "}";
            }

            // inst id
            {
                std::ofstream idf(cols / "inst_id.txt", std::ios::binary);
                idf << inst_id;
            }

            files_.dir = outdir_;
            files_.frame = open_or_throw(cols / "frame.u64");
            files_.ts_ms = open_or_throw(cols / "ts_ms.u64");
            files_.side = open_or_throw(cols / "side.u8");
            files_.level = open_or_throw(cols / "level.u16");
            files_.px = open_or_throw(cols / "px.f64");
            files_.qty = open_or_throw(cols / "qty.f64");
            files_.count = open_or_throw(cols / "count.u32");
        }

        ~ColWriter()
        {
            close();
        }

        ColWriter(const ColWriter &) = delete;
        ColWriter &operator=(const ColWriter &) = delete;

        void append(uint64_t frame, uint64_t ts_ms, uint8_t side, uint16_t level, double px, double qty, uint32_t count)
        {
            write_scalar(files_.frame, frame);
            write_scalar(files_.ts_ms, ts_ms);
            write_scalar(files_.side, side);
            write_scalar(files_.level, level);
            write_scalar(files_.px, px);
            write_scalar(files_.qty, qty);
            write_scalar(files_.count, count);
            ++files_.rows;
        }
        void safe_close(std::FILE *&f)
        {
            if (f)
            {
                std::fclose(f);
                f = nullptr;
            }
        }
        void close()
        {
            if (closed_)
                return;
            safe_close(files_.frame);
            safe_close(files_.ts_ms);
            safe_close(files_.side);
            safe_close(files_.level);
            safe_close(files_.px);
            safe_close(files_.qty);
            safe_close(files_.count);

            // footer
            try
            {
                std::ofstream f(outdir_ / "footer.json", std::ios::binary);
                f << "{\"rows\": " << files_.rows << "}";
            }
            catch (...)
            {
            }

            closed_ = true;
        }

        uint64_t rows() const { return files_.rows; }

    private:
        template <class T>
        static void write_scalar(std::FILE *f, const T &v)
        {
            if (std::fwrite(&v, sizeof(T), 1, f) != 1)
            {
                throw std::runtime_error("colstore write failed");
            }
        }

        fs::path outdir_;
        ColFiles files_{};
        bool closed_ = false;
    };

#ifdef _WIN32
#include <windows.h>
#else
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#endif

    struct MMap
    {
        uint8_t *ptr = nullptr;
        size_t len = 0;
        static MMap map(const fs::path &p)
        {
            MMap m;
            // Portable fallback: read whole file into memory (fast enough for inspection)
            std::ifstream in(p, std::ios::binary);
            if (!in)
                throw std::runtime_error("open failed: " + p.string());
            in.seekg(0, std::ios::end);
            m.len = static_cast<size_t>(in.tellg());
            in.seekg(0);
            m.ptr = static_cast<uint8_t *>(std::malloc(m.len));
            if (!m.ptr)
                throw std::bad_alloc();
            in.read(reinterpret_cast<char *>(m.ptr), m.len);
            return m;
        }
        void unmap()
        {
            if (ptr)
            {
                std::free(ptr);
                ptr = nullptr;
                len = 0;
            }
        }
    };

} // namespace mercury

#endif // MERCURY_COLSTORE_HPP
```

### ndjson_okx.hpp
```cpp
#ifndef MERCURY_NDJSON_OKX_HPP
#define MERCURY_NDJSON_OKX_HPP

#include <string>
#include <string_view>
#include <cstdint>
#include <stdexcept>
#include <simdjson.h>

namespace mercury
{

    // One JSON line -> many rows (one per tuple in bids/asks).
    // on_record signature expected: void(uint64_t ts_ms, uint8_t side, uint16_t level,
    ///                                   double px, double qty, uint32_t count)

    template <class F1, class F2>
    inline void parse_okx_ndjson(const std::string &path, F1 &&on_record, F2 &frame_id, uint64_t &row_limit)
    {
        using namespace simdjson;
        bool continue_parsing = true;
        ondemand::parser parser;
        auto doc_result = padded_string::load(path);
        if (doc_result.error())
        {
            throw std::runtime_error("Failed to load file: " + path);
        }
        padded_string &doc = doc_result.value();

        uint64_t emitted = 0;

        auto stream_res = parser.iterate_many(doc);
        ondemand::document_stream stream = std::move(stream_res).value();
        for (simdjson::ondemand::document_reference d : stream)
        {
            frame_id++; // <- increment per JSON line
            if (d.type().value() != simdjson::ondemand::json_type::object)
                continue;
            auto obj = d.get_object().value();

            // ts may be string or number (OKX gives ms)
            uint64_t ts_ms = 0;

            if (auto ts_it = obj.find_field_unordered("ts"); ts_it.error() == SUCCESS)
            {
                auto v = ts_it.value();
                if (v.type().value() == ondemand::json_type::string)
                {
                    std::string s = std::string(v.get_string().value());
                    ts_ms = static_cast<uint64_t>(std::strtoull(s.c_str(), nullptr, 10));
                }
                else
                {
                    ts_ms = uint64_t(v.get_uint64().value());
                }
            }

            auto maybe_stop = [&](void) -> bool
            {
                if (row_limit == 0)
                    return false;
                return emitted >= row_limit;
            };

            // helper: emit one row per [px, qty, count?] tuple
            auto push_side = [&](ondemand::value arr_val, uint8_t side)
            {
                if (arr_val.type().value() != ondemand::json_type::array)
                    return;
                ondemand::array arr = arr_val.get_array().value();
                uint16_t level = 0;
                for (auto lvl_res : arr)
                {
                    ondemand::value lvl_v = lvl_res.value();
                    if (lvl_v.type().value() != ondemand::json_type::array)
                    {
                        ++level;
                        continue;
                    }
                    ondemand::array triple = lvl_v.get_array().value();

                    double px = 0.0;
                    double qty = 0.0;
                    uint32_t count = 0; 

                    size_t idx = 0;
                    for (auto c_res : triple)
                    {
                        auto c = c_res.value();
                        if (idx == 0)
                        {
                            if (c.type().value() == ondemand::json_type::string)
                            {
                                std::string s = std::string(c.get_string().value());
                                px = std::strtod(s.c_str(), nullptr);
                            }
                            else
                            {
                                px = double(c.get_double().value());
                            }
                        }
                        else if (idx == 1)
                        {
                            if (c.type().value() == ondemand::json_type::string)
                            {
                                std::string s = std::string(c.get_string().value());
                                qty = std::strtod(s.c_str(), nullptr);
                            }
                            else
                            {
                                qty = double(c.get_double().value());
                            }
                        }
                        else if (idx == 2)
                        {
                            if (c.type().value() == ondemand::json_type::string)
                            {
                                std::string s = std::string(c.get_string().value());
                                count = static_cast<uint32_t>(std::strtoul(s.c_str(), nullptr, 10));
                            }
                            else
                            {
                                // Some venues encode counts as integers but ondemand lets us fetch as uint64
                                count = static_cast<uint32_t>(c.get_uint64().value());
                            }
                        }
                        ++idx;
                    }

                    on_record(ts_ms, side, level, px, qty, count);

                    if (maybe_stop())
                    {
                        continue_parsing = false;
                        break;
                    }
                    ++level;
                    ++emitted;
                }
            };

            // bids / b
            if (auto it = obj.find_field_unordered("bids"); it.error() == SUCCESS)
            {
                push_side(it.value(), /*side=*/0);
            }
            else if (auto it2 = obj.find_field_unordered("b"); it2.error() == SUCCESS)
            {
                push_side(it2.value(), /*side=*/0);
            }

            // asks / a
            if (auto it = obj.find_field_unordered("asks"); it.error() == SUCCESS)
            {
                push_side(it.value(), /*side=*/1);
            }
            else if (auto it2 = obj.find_field_unordered("a"); it2.error() == SUCCESS)
            {
                push_side(it2.value(), /*side=*/1);
            }
            if (!continue_parsing)
            {
                break;
            }
        }
    }

} // namespace mercury

#endif // MERCURY_NDJSON_OKX_HPP
```

### okx_to_colstore.cpp
```cpp
#include <iostream>
#include <filesystem>
#include <string>
#include <string_view>
#include <cstdlib>

#include <simdjson.h>
#include "mercury/colstore.hpp"
#include "mercury/ndjson_okx.hpp"

using namespace std;

int main(int argc, char **argv)
{
    if (argc < 4)
    {
        cerr << "Usage: okx_to_colstore <input.data> <out_dir> <instId>\n";
        cerr << "Example: okx_to_colstore BTC-USDT-L2orderbook-400lv-2025-09-20.data out/BTC-USDT BTC-USDT\n";
        return 1;
    }
    const string in_path = argv[1];
    const filesystem::path outdir = argv[2];
    const string inst_id = argv[3];
    uint64_t row_limit = 0;
    if (argc >= 5)
    {
        row_limit = std::strtoull(argv[4], nullptr, 10);
    }

    try
    {
        mercury::ColWriter writer(outdir, inst_id);

        size_t emitted = 0;
        uint64_t frame = 0;

        mercury::parse_okx_ndjson(in_path, [&](uint64_t ts_ms, uint8_t side, uint16_t level, double px, double qty, uint32_t count)
                                  {
                                      if (qty <= 0.0)
                                          return;
                                      writer.append(frame, ts_ms, side, level, px, qty, count);
                                      ++emitted;
                                      if ((emitted % 1'000'000) == 0)
                                      {
                                          cerr << "… " << emitted << " rows written\n";
                                      } }, frame, row_limit);

        writer.close();
        cout << "Wrote " << writer.rows() << " rows to " << outdir << "\n";
    }
    catch (const std::exception &e)
    {
        cerr << "ERROR: " << e.what() << "\n";
        return 2;
    }
    return 0;
}

```

