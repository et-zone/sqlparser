# sqlparser [![Build Status](https://img.shields.io/travis/et-zone/sqlparser.svg)](https://travis-ci.org/et-zone/sqlparser) [![Coverage](https://img.shields.io/coveralls/et-zone/sqlparser.svg)](https://coveralls.io/github/et-zone/sqlparser) [![Report card](https://goreportcard.com/badge/github.com/et-zone/sqlparser)](https://goreportcard.com/report/github.com/et-zone/sqlparser) [![GoDoc](https://godoc.org/github.com/et-zone/sqlparser?status.svg)](https://godoc.org/github.com/et-zone/sqlparser)

Go package for parsing MySQL SQL queries.

## Notice

The backbone of this repo is extracted from [vitessio/vitess](https://github.com/vitessio/vitess).

Inside vitessio/vitess there is a very nicely written sql parser. However as it's not a self-contained application, I created this one.
It applies the same LICENSE as vitessio/vitess.

## Usage

```go
import (
    "github.com/et-zone/sqlparser"
)
```

Then use:

```go
sql := "SELECT * FROM table WHERE a = 'abc'"
stmt, err := sqlparser.Parse(sql)
if err != nil {
	// Do something with the err
}

// Otherwise do something with stmt
switch stmt := stmt.(type) {
case *sqlparser.Select:
	_ = stmt
case *sqlparser.Insert:
}
```

Alternative to read many queries from a io.Reader:

```go
r := strings.NewReader("INSERT INTO table1 VALUES (1, 'a'); INSERT INTO table2 VALUES (3, 4);")

tokens := sqlparser.NewTokenizer(r)
for {
	stmt, err := sqlparser.ParseNext(tokens)
	if err == io.EOF {
		break
	}
	// Do something with stmt or err.
}
```

See [parse_test.go](https://github.com/et-zone/sqlparser/blob/master/parse_test.go) for more examples, or read the [godoc](https://godoc.org/github.com/et-zone/sqlparser).


## Porting Instructions

You only need the below if you plan to try and keep this library up to date with [vitessio/vitess](https://github.com/vitessio/vitess).

### Keeping up to date

```bash
shopt -s nullglob
VITESS=${GOPATH?}/src/vitess.io/vitess/go/
et-zone=${GOPATH?}/src/github.com/et-zone/sqlparser/

# Create patches for everything that changed
LASTIMPORT=1b7879cb91f1dfe1a2dfa06fea96e951e3a7aec5
for path in ${VITESS?}/{vt/sqlparser,sqltypes,bytes2,hack}; do
	cd ${path}
	git format-patch ${LASTIMPORT?} .
done;

# Apply patches to the dependencies
cd ${et-zone?}
git am --directory dependency -p2 ${VITESS?}/{sqltypes,bytes2,hack}/*.patch

# Apply the main patches to the repo
cd ${et-zone?}
git am -p4 ${VITESS?}/vt/sqlparser/*.patch

# If you encounter diff failures, manually fix them with
patch -p4 < .git/rebase-apply/patch
...
git add name_of_files
git am --continue

# Cleanup
rm ${VITESS?}/{sqltypes,bytes2,hack}/*.patch ${VITESS?}/*.patch

# and Finally update the LASTIMPORT in this README.
```

### Fresh install

TODO: Change these instructions to use git to copy the files, that'll make later patching easier.

```bash
VITESS=${GOPATH?}/src/vitess.io/vitess/go/
et-zone=${GOPATH?}/src/github.com/et-zone/sqlparser/

cd ${et-zone?}

# Copy all the code
cp -pr ${VITESS?}/vt/sqlparser/ .
cp -pr ${VITESS?}/sqltypes dependency
cp -pr ${VITESS?}/bytes2 dependency
cp -pr ${VITESS?}/hack dependency

# Delete some code we haven't ported
rm dependency/sqltypes/arithmetic.go dependency/sqltypes/arithmetic_test.go dependency/sqltypes/event_token.go dependency/sqltypes/event_token_test.go dependency/sqltypes/proto3.go dependency/sqltypes/proto3_test.go dependency/sqltypes/query_response.go dependency/sqltypes/result.go dependency/sqltypes/result_test.go

# Some automated fixes

# Fix imports
sed -i '.bak' 's_vitess.io/vitess/go/vt/proto/query_github.com/et-zone/sqlparser/dependency/querypb_g' *.go dependency/sqltypes/*.go
sed -i '.bak' 's_vitess.io/vitess/go/_github.com/et-zone/sqlparser/dependency/_g' *.go dependency/sqltypes/*.go

# Copy the proto, but basically drop everything we don't want
cp -pr ${VITESS?}/vt/proto/query dependency/querypb

sed -i '.bak' 's_.*Descriptor.*__g' dependency/querypb/*.go
sed -i '.bak' 's_.*ProtoMessage.*__g' dependency/querypb/*.go

sed -i '.bak' 's/proto.CompactTextString(m)/"TODO"/g' dependency/querypb/*.go
sed -i '.bak' 's/proto.EnumName/EnumName/g' dependency/querypb/*.go

sed -i '.bak' 's/proto.Equal/reflect.DeepEqual/g' dependency/sqltypes/*.go

# Remove the error library
sed -i '.bak' 's/vterrors.Errorf([^,]*, /fmt.Errorf(/g' *.go dependency/sqltypes/*.go
sed -i '.bak' 's/vterrors.New([^,]*, /errors.New(/g' *.go dependency/sqltypes/*.go
```

### Testing

```bash
VITESS=${GOPATH?}/src/vitess.io/vitess/go/
et-zone=${GOPATH?}/src/github.com/et-zone/sqlparser/

cd ${et-zone?}

# Test, fix and repeat
go test ./...

# Finally make some diffs (for later reference)
diff -u ${VITESS?}/sqltypes/        ${et-zone?}/dependency/sqltypes/ > ${et-zone?}/patches/sqltypes.patch
diff -u ${VITESS?}/bytes2/          ${et-zone?}/dependency/bytes2/   > ${et-zone?}/patches/bytes2.patch
diff -u ${VITESS?}/vt/proto/query/  ${et-zone?}/dependency/querypb/  > ${et-zone?}/patches/querypb.patch
diff -u ${VITESS?}/vt/sqlparser/    ${et-zone?}/                     > ${et-zone?}/patches/sqlparser.patch
```