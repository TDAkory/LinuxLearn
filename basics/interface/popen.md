# [popen](https://man7.org/linux/man-pages/man3/popen.3.html)

The `popen()` function opens a process by creating a pipe, forking,
and invoking the shell.  Since a pipe is by definition
unidirectional, the type argument may specify only reading or
writing, not both; the resulting stream is correspondingly read-
only or write-only.
The command argument is a pointer to a null-terminated string
containing a shell command line.  This command is passed to
/bin/sh using the -c flag; interpretation, if any, is performed
by the shell.

```cpp
// Get a result of some script by exec and print
std::string GetVolcanoHostIP() {
    std::array<char, 128> buffer;
    std::string result;
    std::shared_ptr<FILE> pipe(popen(inject_host_ip_path, "r"), pclose);
    if (!pipe) {
        // Do something
        return result;
    }
    while (!feof(pipe.get())) {
        if (fgets(buffer.data(), 128, pipe.get())) {
            result += buffer.data();
        }
    }
    boost::algorithm::trim_if(result, boost::is_any_of("\n"));
    return result;
}
```