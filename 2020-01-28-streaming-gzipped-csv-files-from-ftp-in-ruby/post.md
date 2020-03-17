---
title: Using Ruby Enumerators for streaming big gzipped CSV files from FTP
published: true
description: Check out how I managed to make use of lazy enumerators to FTP stream, ungzip and process a huge CSV file line by line.
tags: ruby, ftp, gzip, enumerable
---

Today I'd like to share with you a solution to a problem that gave me some headache recently, so you can spend your time on something more interesting (have a ☕ or something).

# The task

The goal was to write a rake task that'd:
1. Fetch a big (1GB [gzip](https://en.wikipedia.org/wiki/Gzip)ped) CSV file from FTP
2. Ungzip it (18GB ungzipped)
3. Parse each row
4. Insert it into an SQL database

I'll describe my approach to steps 1-3. Point 4 will is a material for another story 😅

# The solution

Of course loading 18 gigabyte file into the memory wasn't the best idea, so I decided that I'll try streaming the file in chunks from FTP, ungzip each chunk and then collect the full lines. This should be more merciful to RAM usage and would e.g. allow reading a few records from the file without having to load the whole file first.

Fortunately this approach is possible, because:
1. Net::FTP library supports streaming the file in chunks ([getbinaryfile](https://docs.ruby-lang.org/en/2.0.0/Net/FTP.html#method-i-getbinaryfile) method).
2. [Zlib](https://ruby-doc.org/stdlib-2.6.5/libdoc/zlib/rdoc/Zlib.html) library supports buffered ungzipping

Ok, so how do I glue it all together using the ruby code? It turns out that [Enumerator](https://ruby-doc.org/core-2.6/Enumerator.html) and [Enumerator::Lazy](https://ruby-doc.org/core-2.5.0/Enumerator/Lazy.html) come in handy here...

### Solution code

Let's take a look at the full solution and analyze it step by step:

```ruby
# frozen_string_literal: true

require "net/ftp"

class Loader
  def load
    Enumerator.new { |main_enum| stream(main_enum) }
  end

  private

  attr_reader :ftp, :inflater

  def stream(main_enum)
    init_ftp_connection
    init_gzip_inflater

    # Drop the header line
    split_lines.lazy.drop(1).each { |line| main_enum << preprocess_row(line) }
  ensure
    ftp&.close
    inflater&.close
  end

  def split_lines
    buffer = ""

    Enumerator.new do |yielder|
      ungzip.each do |decompressed_chunk|
        buffer += decompressed_chunk
        new_buffer = ""
        buffer.each_line do |l|
          l.ends_with?("\n") ? yielder << l : new_buffer += l
        end

        buffer = new_buffer
      end
    end
  end

  def ungzip
    Enumerator.new do |yielder|
      stream_file_from_ftp.each do |compressed|
        inflater.inflate(compressed) do |decompressed_chunk|
          yielder << decompressed_chunk
        end
      end
    end
  end

  def stream_file_from_ftp
    chunk_size = 1024

    Enumerator.new do |ftp_stream_enum|
      ftp.getbinaryfile("file.csv.gz", nil, chunk_size) do |chunk|
        ftp_stream_enum << chunk
      end
    end
  end

  def init_ftp_connection
    @ftp = Net::FTP.new(
      "ftp://host.com",
      "user",
      "pass"
    ).tap { |f| f.passive = true }
  end

  def init_gzip_inflater
    # Taken from examples in:
    # https://docs.ruby-lang.org/en/2.0.0/Zlib/Inflate.html
    @inflater = Zlib::Inflate.new(Zlib::MAX_WBITS + 32)
  end

  def preprocess_row(row)
    row.chomp.gsub('"', "").split(",")
  end
end

```

The `Loader` class is using a set of nested `Enumerator`s:
- `stream_file_from_ftp` - loads a chunk of file from ftp and yields it to the next enumerator.
- `ungzip` - gathers loaded FTP chunks into decompressable gzip blocks, decompresses them and yields such decoded chunks to the next enumerator.
- `split_lines` - collects decompressed gzip blocks, splits them by the newline character and yields full lines to the next enumerator.
- `main_enum` - preprocesses each loaded CSV line by removing quotation marks and splitting by comma signs.

Note that we called `lazy` on `split_lines` enumerator. This is the most important part of this solution - it allows e.g. loading the first row without having to download and ungzip the full file, but only the part necessary for loading one row. Without it our program would load all rows into the memory only to return the first of them.

# Summary
I recommend you to remember the presented pattern of combining `Enumerator`s and using the lazy enumeration method. It will come handy anytime when writing code that processes streamable data.


# But wait there's more...
The solution I presented has one drawback - it keeps a TCP socket connection when processing consequent rows. This may sometimes cause a socket time out when processing really huge files.

In this case I'd suggest downloading the file to disk before processing. This can be achieved using Ruby's [OpenURI](https://ruby-doc.org/stdlib-2.6.3/libdoc/open-uri/rdoc/OpenURI.html) library which supports FTP protocol.

Feel free to use the code below:
```ruby
class Loader
  def load
    Enumerator.new { |main_enum| stream(main_enum) }
  end

  private

  def stream(main_enum)
    reader = nil
    file_uri.open do |file|
      reader = Zlib::GzipReader.new(file)
      reader.each_line.lazy.drop(1).each do |line|
        main_enum << preprocess_row(line)
      end
    end
  ensure
    reader&.close
  end

  def file_uri
    URI.parse("ftp://user:password@host.com/file.csv.gz")
  end

  def preprocess_row(row)
    row.chomp.gsub('"', "").split(",")
  end
end
```
