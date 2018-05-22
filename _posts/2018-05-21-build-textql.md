---
layout: post
title: Build TextQL under Ubuntu 16.04
tags: textql
comment: true
image: 
published: true
---
# Build TextQL under Ubuntu 16.04

[TextQL](https://github.com/dinedal/textql) is Golang-based console command, which can easily execute SQL against structured text like CSV or TSV.

## Step 1: Install Golang development environment and SQLite3

```console
sudo apt-get install -y golang-1.10
sudo apt-get install -y sqlite3
```

Note: If you do not install SQLite3, step 3 build will fail with error

```console
go test ./storage/

[/tmp/textql_test284227772 pragma integrity_check;]
--- FAIL: TestSQLiteStorageSaveTo (0.00s)
        sqlite_test.go:83: exec: "sqlite3": executable file not found in $PATH
FAIL
FAIL    github.com/dinedal/textql/storage       0.012s
Makefile:32: recipe for target 'test' failed
make: *** [test] Error 1
```

## Step 2: Download and install packages and dependencies

```bash
export GOROOT="/usr/lib/go-1.10"         # golang-1.10 installation directory
export GOPATH="/home/vagrant/go"         # goland workspace directory

go get -u github.com/dinedal/textql/...
```

It downloads code of both TextQL and its dependencies under ${GOPATH}

```console
${GOPATH}/src/github.com/dinedal/textql
${GOPATH}/src/github.com/dinedal/textql/src/github.com/mattn/go-runewidth
${GOPATH}/src/github.com/dinedal/textql/src/github.com/mattn/go-sqlite3
${GOPATH}/src/github.com/dinedal/textql/src/github.com/olekukonko/tablewriter
```

It also downloads the TextQL console command binary.

```console
${GOPATH}/src/github.com/dinedal/textql/bin/textql
```

## Step 3: Build from source

Build pulls dependency to vendor/ directory. Build result is build/textql.

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ export GOROOT="/usr/lib/go-1.10"
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ export GOPATH="/home/vagrant/go"
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ make
curl -L https://github.com/Masterminds/glide/releases/download/0.5.0/glide-linux-386.zip -o glide.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   609    0   609    0     0    322      0 --:--:--  0:00:01 --:--:--   322
100 2751k  100 2751k    0     0   165k      0  0:00:16  0:00:16 --:--:--  350k
unzip glide.zip
Archive:  glide.zip
   creating: linux-386/
  inflating: linux-386/glide
  inflating: linux-386/LICENSE.txt
  inflating: linux-386/README.md
mv ./linux-386/glide ./glide
rm -fr ./linux-386
rm ./glide.zip
./glide install
[WARN] You must install the Go 1.5 or greater toolchain to work with Glide.
[WARN] To use Glide, you must set GO15VENDOREXPERIMENT=1
[INFO] Fetching updates for github.com/mattn/go-sqlite3.
[INFO] Fetching updates for github.com/olekukonko/tablewriter.
[INFO] Fetching updates for github.com/mattn/go-runewidth.
[INFO] Setting version for github.com/mattn/go-sqlite3.
[INFO] Setting version for github.com/olekukonko/tablewriter.
[INFO] Setting version for github.com/mattn/go-runewidth.
[INFO] Checking dependencies for updates. Godeps: false, GPM: false
[INFO] Inspecting /home/vagrant/go/src/github.com/dinedal/textql/vendor.
[INFO] Looking in /home/vagrant/go/src/github.com/dinedal/textql/vendor/github.com/mattn/go-sqlite3 for a glide.yaml file.
[INFO] Package github.com/mattn/go-sqlite3 manages its own dependencies.
[INFO] Looking in /home/vagrant/go/src/github.com/dinedal/textql/vendor/github.com/olekukonko/tablewriter for a glide.yaml file.
[INFO] Package github.com/olekukonko/tablewriter manages its own dependencies.
[INFO] Looking in /home/vagrant/go/src/github.com/dinedal/textql/vendor/github.com/mattn/go-runewidth for a glide.yaml file.
[INFO] Package github.com/mattn/go-runewidth manages its own dependencies.
go test ./inputs/
ok      github.com/dinedal/textql/inputs        (cached)
go test ./storage/
ok      github.com/dinedal/textql/storage       (cached)
go build -ldflags "-X main.VERSION=`cat VERSION`" -o ./build/textql ./textql/main.go
```

## Step 4: Test with TextQL

Sample csv file

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ cat vendor/github.com/olekukonko/tablewriter/test.csv
first_name,last_name,ssn
John,Barry,123456
Kathy,Smith,687987
Bob,McCornick,3979870
```

Query with SQL

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ build/textql -header -sql 'SELECT * FROM test WHERE first_name LIKE "J%"' vendor/github.com/olekukonko/tablewriter/test.csv
John,Barry,123456
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ build/textql -header -sql 'SELECT * FROM test WHERE ssn>200000' vendor/github.com/olekukonko/tablewriter/test.csv
Kathy,Smith,687987
Bob,McCornick,3979870
```

Query in SQLite3 console

```console
vagrant@k8smaster:~/go/src/github.com/dinedal/textql$ build/textql -header -console vendor/github.com/olekukonko/tablewriter/test.csv
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> SELECT * FROM test;
John|Barry|123456
Kathy|Smith|687987
Bob|McCornick|3979870
sqlite> SELECT last_name FROM test WHERE first_name="John";
Barry
sqlite>
2018/05/22 02:32:33 exit status 1
```
