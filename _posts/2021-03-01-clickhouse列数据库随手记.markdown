---

layout:     post
title:      "clickhouse随手记录"
subtitle:   " \"clickhouse随手记录查询分析\""
date:       2021-03-01 21:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
抽样查询分析
ClickHouse 运行允许分析查询执行的采样分析器。使用探查器，您可以找到在查询执行期间使用最频繁的源代码例程。您可以跟踪 CPU 时间和挂钟时间，包括空闲时间。

使用探查器：

    设置服务器配置的trace_log部分。

    此部分配置包含分析器运行结果的trace_log系统表。它是默认配置的。请记住，此表中的数据仅对正在运行的服务器有效。服务器重启后，ClickHouse 不会清理表，所有存储的虚拟内存地址可能会失效。

    设置query_profiler_cpu_time_period_ns或query_profiler_real_time_period_ns设置。这两种设置可以同时使用。

    这些设置允许您配置分析器计时器。由于这些是会话设置，因此您可以为整个服务器、单个用户或用户配置文件、交互式会话以及每个单独的查询获得不同的采样频率。

    默认采样频率为每秒一个样本，同时启用 CPU 和实时计时器。此频率允许收集有关 ClickHouse 集群的足够信息。同时，在此频率下，分析器不会影响 ClickHouse 服务器的性能。如果您需要分析每个单独的查询，请尝试使用更高的采样频率。

分析trace_log系统表：

    安装clickhouse-common-static-dbg软件包。请参阅从 DEB 包安装。

    通过allow_introspection_functions设置允许内省函数。

    出于安全原因，默认情况下禁用自省功能。

    使用addressToLine,addressToSymbol和demangle introspection 函数获取函数名称及其在 ClickHouse 代码中的位置。要获取某个查询的配置文件，您需要聚合trace_log表中的数据。您可以按单个函数或整个堆栈跟踪聚合数据。

如果您需要可视化trace_log信息，请尝试flamegraph和speedscope。

例子：

    trace_log按查询标识符和当前日期过滤数据。

    通过堆栈跟踪聚合。

    使用自省功能，我们将获得以下报告：

    符号名称和相应的源代码函数。
    这些函数的源代码位置。
    SELECT
        count(),
        arrayStringConcat(arrayMap(x -> concat(demangle(addressToSymbol(x)), '\n    ', addressToLine(x)), trace), '\n') AS sym
    FROM system.trace_log
    WHERE (query_id = 'ebca3574-ad0a-400a-9cbc-dca382f5998c') AND (event_date = today())
    GROUP BY trace
    ORDER BY count() DESC
    LIMIT 10
    Row 1:
    ──────
    count(): 6344
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    read

    DB::ReadBufferFromFileDescriptor::nextImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/ReadBufferFromFileDescriptor.cpp:56
    DB::CompressedReadBufferBase::readCompressedData(unsigned long&, unsigned long&)
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/ReadBuffer.h:54
    DB::CompressedReadBufferFromFile::nextImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Compression/CompressedReadBufferFromFile.cpp:22
    DB::CompressedReadBufferFromFile::seek(unsigned long, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/Compression/CompressedReadBufferFromFile.cpp:63
    DB::MergeTreeReaderStream::seekToMark(unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReaderStream.cpp:200
    std::_Function_handler<DB::ReadBuffer* (std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&), DB::MergeTreeReader::readData(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, DB::IColumn&, unsigned long, bool, unsigned long, bool)::{lambda(bool)#1}::operator()(bool) const::{lambda(std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&)#1}>::_M_invoke(std::_Any_data const&, std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:212
    DB::IDataType::deserializeBinaryBulkWithMultipleStreams(DB::IColumn&, unsigned long, DB::IDataType::DeserializeBinaryBulkSettings&, std::shared_ptr<DB::IDataType::DeserializeBinaryBulkState>&) const
        /usr/local/include/c++/9.1.0/bits/std_function.h:690
    DB::MergeTreeReader::readData(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, DB::IColumn&, unsigned long, bool, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:232
    DB::MergeTreeReader::readRows(unsigned long, bool, unsigned long, DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:111
    DB::MergeTreeRangeReader::DelayedStream::finalize(DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:35
    DB::MergeTreeRangeReader::continueReadingChain(DB::MergeTreeRangeReader::ReadResult&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:219
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:487
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 2:
    ──────
    count(): 3295
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    __pthread_cond_wait

    std::condition_variable::wait(std::unique_lock<std::mutex>&)
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/src/c++11/../../../../../gcc-9.1.0/libstdc++-v3/src/c++11/condition_variable.cc:55
    Poco::Semaphore::wait()
        /home/milovidov/ClickHouse/build_gcc9/../contrib/poco/Foundation/src/Semaphore.cpp:61
    DB::UnionBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/x86_64-pc-linux-gnu/bits/gthr-default.h:748
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Core/Block.h:90
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::LimitBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::AsynchronousBlockInputStream::calculate()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    std::_Function_handler<void (), DB::AsynchronousBlockInputStream::next()::{lambda()#1}>::_M_invoke(std::_Any_data const&)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:551
    ThreadPoolImpl<ThreadFromGlobalPool>::worker(std::_List_iterator<ThreadFromGlobalPool>)
        /usr/local/include/c++/9.1.0/x86_64-pc-linux-gnu/bits/gthr-default.h:748
    ThreadFromGlobalPool::ThreadFromGlobalPool<ThreadPoolImpl<ThreadFromGlobalPool>::scheduleImpl<void>(std::function<void ()>, int, std::optional<unsigned long>)::{lambda()#3}>(ThreadPoolImpl<ThreadFromGlobalPool>::scheduleImpl<void>(std::function<void ()>, int, std::optional<unsigned long>)::{lambda()#3}&&)::{lambda()#1}::operator()() const
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/ThreadPool.h:146
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 3:
    ──────
    count(): 1978
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    DB::VolnitskyBase<true, true, DB::StringSearcher<true, true> >::search(unsigned char const*, unsigned long) const
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::MatchImpl<true, false>::vector_constant(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::FunctionsStringSearch<DB::MatchImpl<true, false>, DB::NameLike>::executeImpl(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::PreparedFunctionImpl::execute(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Functions/IFunction.cpp:464
    DB::ExpressionAction::execute(DB::Block&, bool) const
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:677
    DB::ExpressionActions::execute(DB::Block&, bool) const
        /home/milovidov/ClickHouse/build_gcc9/../src/Interpreters/ExpressionActions.cpp:739
    DB::MergeTreeRangeReader::executePrewhereActionsAndFilterColumns(DB::MergeTreeRangeReader::ReadResult&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:660
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:546
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 4:
    ──────
    count(): 1913
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    DB::VolnitskyBase<true, true, DB::StringSearcher<true, true> >::search(unsigned char const*, unsigned long) const
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::MatchImpl<true, false>::vector_constant(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::FunctionsStringSearch<DB::MatchImpl<true, false>, DB::NameLike>::executeImpl(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::PreparedFunctionImpl::execute(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Functions/IFunction.cpp:464
    DB::ExpressionAction::execute(DB::Block&, bool) const
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:677
    DB::ExpressionActions::execute(DB::Block&, bool) const
        /home/milovidov/ClickHouse/build_gcc9/../src/Interpreters/ExpressionActions.cpp:739
    DB::MergeTreeRangeReader::executePrewhereActionsAndFilterColumns(DB::MergeTreeRangeReader::ReadResult&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:660
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:546
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 5:
    ──────
    count(): 1672
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    DB::VolnitskyBase<true, true, DB::StringSearcher<true, true> >::search(unsigned char const*, unsigned long) const
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::MatchImpl<true, false>::vector_constant(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::FunctionsStringSearch<DB::MatchImpl<true, false>, DB::NameLike>::executeImpl(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::PreparedFunctionImpl::execute(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Functions/IFunction.cpp:464
    DB::ExpressionAction::execute(DB::Block&, bool) const
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:677
    DB::ExpressionActions::execute(DB::Block&, bool) const
        /home/milovidov/ClickHouse/build_gcc9/../src/Interpreters/ExpressionActions.cpp:739
    DB::MergeTreeRangeReader::executePrewhereActionsAndFilterColumns(DB::MergeTreeRangeReader::ReadResult&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:660
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:546
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 6:
    ──────
    count(): 1531
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    read

    DB::ReadBufferFromFileDescriptor::nextImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/ReadBufferFromFileDescriptor.cpp:56
    DB::CompressedReadBufferBase::readCompressedData(unsigned long&, unsigned long&)
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/ReadBuffer.h:54
    DB::CompressedReadBufferFromFile::nextImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Compression/CompressedReadBufferFromFile.cpp:22
    void DB::deserializeBinarySSE2<4>(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::ReadBuffer&, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/ReadBuffer.h:53
    DB::DataTypeString::deserializeBinaryBulk(DB::IColumn&, DB::ReadBuffer&, unsigned long, double) const
        /home/milovidov/ClickHouse/build_gcc9/../src/DataTypes/DataTypeString.cpp:202
    DB::MergeTreeReader::readData(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, DB::IColumn&, unsigned long, bool, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:232
    DB::MergeTreeReader::readRows(unsigned long, bool, unsigned long, DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:111
    DB::MergeTreeRangeReader::DelayedStream::finalize(DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:35
    DB::MergeTreeRangeReader::startReadingChain(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:219
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 7:
    ──────
    count(): 1034
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    DB::VolnitskyBase<true, true, DB::StringSearcher<true, true> >::search(unsigned char const*, unsigned long) const
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::MatchImpl<true, false>::vector_constant(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul> const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::FunctionsStringSearch<DB::MatchImpl<true, false>, DB::NameLike>::executeImpl(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long)
        /opt/milovidov/ClickHouse/build_gcc9/programs/clickhouse
    DB::PreparedFunctionImpl::execute(DB::Block&, std::vector<unsigned long, std::allocator<unsigned long> > const&, unsigned long, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Functions/IFunction.cpp:464
    DB::ExpressionAction::execute(DB::Block&, bool) const
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:677
    DB::ExpressionActions::execute(DB::Block&, bool) const
        /home/milovidov/ClickHouse/build_gcc9/../src/Interpreters/ExpressionActions.cpp:739
    DB::MergeTreeRangeReader::executePrewhereActionsAndFilterColumns(DB::MergeTreeRangeReader::ReadResult&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:660
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:546
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 8:
    ──────
    count(): 989
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    __lll_lock_wait

    pthread_mutex_lock

    DB::MergeTreeReaderStream::loadMarks()
        /usr/local/include/c++/9.1.0/bits/std_mutex.h:103
    DB::MergeTreeReaderStream::MergeTreeReaderStream(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> > const&, DB::MarkCache*, bool, DB::UncompressedCache*, unsigned long, unsigned long, unsigned long, DB::MergeTreeIndexGranularityInfo const*, std::function<void (DB::ReadBufferFromFileBase::ProfileInfo)> const&, int)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReaderStream.cpp:107
    std::_Function_handler<void (std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&), DB::MergeTreeReader::addStreams(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, std::function<void (DB::ReadBufferFromFileBase::ProfileInfo)> const&, int)::{lambda(std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&)#1}>::_M_invoke(std::_Any_data const&, std::vector<DB::IDataType::Substream, std::allocator<DB::IDataType::Substream> > const&)
        /usr/local/include/c++/9.1.0/bits/unique_ptr.h:147
    DB::MergeTreeReader::addStreams(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, std::function<void (DB::ReadBufferFromFileBase::ProfileInfo)> const&, int)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:677
    DB::MergeTreeReader::MergeTreeReader(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, std::shared_ptr<DB::MergeTreeDataPart const> const&, DB::NamesAndTypesList const&, DB::UncompressedCache*, DB::MarkCache*, bool, DB::MergeTreeData const&, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> > const&, unsigned long, unsigned long, std::map<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, double, std::less<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >, std::allocator<std::pair<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const, double> > > const&, std::function<void (DB::ReadBufferFromFileBase::ProfileInfo)> const&, int)
        /usr/local/include/c++/9.1.0/bits/stl_list.h:303
    DB::MergeTreeThreadSelectBlockInputStream::getNewTask()
        /usr/local/include/c++/9.1.0/bits/std_function.h:259
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:54
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 9:
    ───────
    count(): 779
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    void DB::deserializeBinarySSE2<4>(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::ReadBuffer&, unsigned long)
        /usr/local/lib/gcc/x86_64-pc-linux-gnu/9.1.0/include/emmintrin.h:727
    DB::DataTypeString::deserializeBinaryBulk(DB::IColumn&, DB::ReadBuffer&, unsigned long, double) const
        /home/milovidov/ClickHouse/build_gcc9/../src/DataTypes/DataTypeString.cpp:202
    DB::MergeTreeReader::readData(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, DB::IColumn&, unsigned long, bool, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:232
    DB::MergeTreeReader::readRows(unsigned long, bool, unsigned long, DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:111
    DB::MergeTreeRangeReader::DelayedStream::finalize(DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:35
    DB::MergeTreeRangeReader::startReadingChain(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:219
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone


    Row 10:
    ───────
    count(): 666
    sym:     StackTrace::StackTrace(ucontext_t const&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Common/StackTrace.cpp:208
    DB::(anonymous namespace)::writeTraceInfo(DB::TimerType, int, siginfo_t*, void*) [clone .isra.0]
        /home/milovidov/ClickHouse/build_gcc9/../src/IO/BufferBase.h:99


    void DB::deserializeBinarySSE2<4>(DB::PODArray<unsigned char, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::PODArray<unsigned long, 4096ul, AllocatorWithHint<false, AllocatorHints::DefaultHint, 67108864ul>, 15ul, 16ul>&, DB::ReadBuffer&, unsigned long)
        /usr/local/lib/gcc/x86_64-pc-linux-gnu/9.1.0/include/emmintrin.h:727
    DB::DataTypeString::deserializeBinaryBulk(DB::IColumn&, DB::ReadBuffer&, unsigned long, double) const
        /home/milovidov/ClickHouse/build_gcc9/../src/DataTypes/DataTypeString.cpp:202
    DB::MergeTreeReader::readData(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, DB::IDataType const&, DB::IColumn&, unsigned long, bool, unsigned long, bool)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:232
    DB::MergeTreeReader::readRows(unsigned long, bool, unsigned long, DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeReader.cpp:111
    DB::MergeTreeRangeReader::DelayedStream::finalize(DB::Block&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:35
    DB::MergeTreeRangeReader::startReadingChain(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeRangeReader.cpp:219
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeRangeReader::read(unsigned long, std::vector<DB::MarkRange, std::allocator<DB::MarkRange> >&)
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::MergeTreeBaseSelectBlockInputStream::readFromPartImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/Storages/MergeTree/MergeTreeBaseSelectBlockInputStream.cpp:158
    DB::MergeTreeBaseSelectBlockInputStream::readImpl()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ExpressionBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ExpressionBlockInputStream.cpp:34
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::PartialSortingBlockInputStream::readImpl()
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/PartialSortingBlockInputStream.cpp:13
    DB::IBlockInputStream::read()
        /usr/local/include/c++/9.1.0/bits/stl_vector.h:108
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::loop(unsigned long)
        /usr/local/include/c++/9.1.0/bits/atomic_base.h:419
    DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::thread(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long)
        /home/milovidov/ClickHouse/build_gcc9/../src/DataStreams/ParallelInputsProcessor.h:215
    ThreadFromGlobalPool::ThreadFromGlobalPool<void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*, std::shared_ptr<DB::ThreadGroupStatus>, unsigned long&>(void (DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>::*&&)(std::shared_ptr<DB::ThreadGroupStatus>, unsigned long), DB::ParallelInputsProcessor<DB::UnionBlockInputStream::Handler>*&&, std::shared_ptr<DB::ThreadGroupStatus>&&, unsigned long&)::{lambda()#1}::operator()() const
        /usr/local/include/c++/9.1.0/bits/shared_ptr_base.h:729
    ThreadPoolImpl<std::thread>::worker(std::_List_iterator<std::thread>)
        /usr/local/include/c++/9.1.0/bits/unique_lock.h:69
    execute_native_thread_routine
        /home/milovidov/ClickHouse/ci/workspace/gcc/gcc-build/x86_64-pc-linux-gnu/libstdc++-v3/include/bits/unique_ptr.h:81
    start_thread

    __clone
