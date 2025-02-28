
UNAME_S := $(shell uname -s)
UNAME_MS := $(word 2, $(shell uname -ms))

ifneq (,$(wildcard /etc/system-release))
	CENTOSVER := $(shell rpm --eval %rhel)
else
	CENTOSVER := Unknown
endif

IS_ARM = $(if $(findstring aarch64, $(arch)),true,false)
IS_CENTOS = $(if $(findstring Unknown, $(CENTOSVER)),false,true)

# to compile libmariadb_client with support for valgrind enabled, run:
# export USEVALGRIND=1
#
USEVALGRIND := $(shell echo $(USEVALGRIND))

PROXYDEBUG := $(shell echo $(PROXYDEBUG))
ifeq ($(PROXYDEBUG),1)
MYCFLAGS=-O0
MYJEOPT=--enable-xmalloc --enable-prof --enable-fill --enable-debug

ifeq ($(and $(IS_ARM),$(IS_CENTOS)),true)
	MYJEOPT+=--with-lg-page=16
endif

else
MYCFLAGS=-O2
MYJEOPT=--enable-xmalloc --enable-prof

ifeq ($(and $(IS_ARM),$(IS_CENTOS)),true)
	MYJEOPT+=--with-lg-page=16
endif

endif

PROXYSQLCLICKHOUSE := $(shell echo $(PROXYSQLCLICKHOUSE))
ifeq ($(PROXYSQLCLICKHOUSE),1)
default: libconfig libdaemon jemalloc mariadb_client re2 sqlite3 pcre clickhouse-cpp lz4 cityhash microhttpd curl ev libssl libhttpserver libinjection prometheus-cpp
else
default: libconfig libdaemon jemalloc mariadb_client re2 sqlite3 pcre microhttpd curl ev libssl libhttpserver libinjection prometheus-cpp
endif
.PHONY: default


libinjection/libinjection/src/libinjection.a:
	cd libinjection && rm -rf libinjection-*/ || true
	cd libinjection && tar -zxf libinjection-3.10.0.tar.gz
ifneq ($(CENTOSVER),6)
	cd libinjection/libinjection && patch -p1 < ../update-build-py3.diff
endif
	sed -i 's/CC=/CC?=/' libinjection/libinjection/src/Makefile
	cd libinjection/libinjection && CC=${CC} CXX=${CXX} ${MAKE}
libinjection: libinjection/libinjection/src/libinjection.a

libssl/openssl/libssl.a:
	cd libssl && rm -rf openssl-openssl-*/ || true
	cd libssl && tar -zxf openssl-*.tar.gz
	cd libssl/openssl && patch crypto/ec/curve448/curve448.c < ../curve448.c-multiplication-overflow.patch
	cd libssl/openssl && patch crypto/asn1/a_time.c < ../a_time.c-multiplication-overflow.patch
	cd libssl/openssl && ./config no-ssl3 no-tests
	cd libssl/openssl && CC=${CC} CXX=${CXX} ${MAKE}
	cd libssl/openssl && ln -s . lib # curl wants this path
libssl: libssl/openssl/libssl.a

GCC_VERSION := $(shell gcc -dumpversion)
MIN_VERSION := 4.9.0
REQUIRE_PATCH = false

ifeq ($(MIN_VERSION),$(lastword $(sort $(GCC_VERSION) $(MIN_VERSION))))
	REQUIRE_PATCH = true
endif

libhttpserver/libhttpserver/build/src/.libs/libhttpserver.a: libmicrohttpd/libmicrohttpd/src/microhttpd/.libs/libmicrohttpd.a
	cd libhttpserver && rm -rf libhttpserver-*/ || true
	cd libhttpserver && tar -zxf libhttpserver-0.18.1.tar.gz
ifeq ($(REQUIRE_PATCH), true)
	cd libhttpserver/libhttpserver && patch src/httpserver/basic_auth_fail_response.hpp < ../basic_auth_fail_response.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/create_webserver.hpp < ../create_webserver.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/deferred_response.hpp < ../deferred_response.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/digest_auth_fail_response.hpp < ../digest_auth_fail_response.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/file_response.hpp < ../file_response.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/http_request.hpp < ../http_request.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/http_resource.hpp < ../http_resource.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/http_response.hpp < ../http_response.hpp.patch
	cd libhttpserver/libhttpserver && patch src/httpserver/string_response.hpp < ../string_response.hpp.patch
	cd libhttpserver/libhttpserver && patch -p0 < ../re2_regex.patch
	cd libhttpserver/libhttpserver && patch -p0 < ../final_val_post_process.patch
endif
ifeq ($(UNAME_S),FreeBSD)
	sed -i -e 's/\/bin\/bash/\/usr\/local\/bin\/bash/' libhttpserver/libhttpserver/bootstrap
endif
	cd libhttpserver/libhttpserver && ./bootstrap && mkdir build
	cd libhttpserver/libhttpserver/build && LDFLAGS=-L$(shell pwd)/libmicrohttpd/libmicrohttpd/src/microhttpd/.libs/ CPPFLAGS=-I$(shell pwd)/libmicrohttpd/libmicrohttpd/src/include ../configure --disable-doxygen-doc --disable-doxygen-dot --disable-doxygen-man --disable-doxygen-html --enable-fastopen=false
	cd libhttpserver/libhttpserver/build && CC=${CC} CXX=${CXX} ${MAKE}
libhttpserver: libhttpserver/libhttpserver/build/src/.libs/libhttpserver.a

libev/libev/.libs/libev.a:
	cd libev && rm -rf libev-*/ || true
	cd libev && tar -zxf libev-4.24.tar.gz
	cd libev/libev && patch ev.c < ../ev.c-multiplication-overflow.patch
	cd libev/libev  && ./configure
	cd libev/libev && CC=${CC} CXX=${CXX} ${MAKE}
ev: libev/libev/.libs/libev.a

curl/curl/lib/.libs/libcurl.a: libssl/openssl/libssl.a
	cd curl && rm -rf curl-*/ || true
	cd curl && tar -zxf curl-*.tar.gz
	#cd curl/curl && ./configure --disable-debug --disable-ftp --disable-ldap --disable-ldaps --disable-rtsp --disable-proxy --disable-dict --disable-telnet --disable-tftp --disable-pop3 --disable-imap --disable-smb --disable-smtp --disable-gopher --disable-manual --disable-ipv6 --disable-sspi --disable-crypto-auth --disable-ntlm-wb --disable-tls-srp --without-nghttp2 --without-libidn2 --without-libssh2 --without-brotli --with-ssl=$(shell pwd)/../../libssl/openssl/ && CC=${CC} CXX=${CXX} ${MAKE}
	cd curl/curl && CFLAGS=-fPIC ./configure --disable-debug --disable-ftp --disable-ldap --disable-ldaps --disable-rtsp --disable-proxy --disable-dict --disable-telnet --disable-tftp --disable-pop3 --disable-imap --disable-smb --disable-smtp --disable-gopher --disable-manual --disable-ipv6 --disable-sspi --disable-ntlm-wb --disable-tls-srp --without-nghttp2 --without-libidn2 --without-libssh2 --without-brotli --without-librtmp --without-libpsl --without-zstd --with-ssl=$(shell pwd)/libssl/openssl/ --enable-shared=no && CC=${CC} CXX=${CXX} ${MAKE}
curl: curl/curl/lib/.libs/libcurl.a

libmicrohttpd/libmicrohttpd/src/microhttpd/.libs/libmicrohttpd.a:
	cd libmicrohttpd && rm -rf libmicrohttpd-*/ || true
	cd libmicrohttpd && rm -f libmicrohttpd || true
ifeq ($(CENTOSVER),6)
	cd libmicrohttpd && ln -s libmicrohttpd-0.9.55 libmicrohttpd
	cd libmicrohttpd && tar -zxf libmicrohttpd-0.9.55.tar.gz
else
	cd libmicrohttpd && ln -s libmicrohttpd-0.9.68 libmicrohttpd
	cd libmicrohttpd && tar -zxf libmicrohttpd-0.9.68.tar.gz
	cd libmicrohttpd/libmicrohttpd && patch src/microhttpd/connection.c < ../connection.c-snprintf-overflow.patch
endif
ifeq ($(OS),Darwin)
	cd libmicrohttpd/libmicrohttpd && patch src/microhttpd/mhd_sockets.c < ../mhd_sockets.c-issue-5977.patch
endif
	cd libmicrohttpd/libmicrohttpd && ./configure --enable-https && CC=${CC} CXX=${CXX} ${MAKE}
microhttpd: libmicrohttpd/libmicrohttpd/src/microhttpd/.libs/libmicrohttpd.a

cityhash/cityhash/src/.libs/libcityhash.a:
	cd cityhash && rm -rf cityhash/ || true
	cd cityhash && tar -zxf cityhash.tar.gz
	cd cityhash/cityhash && cp ../config.guess . && chmod +x config.guess && ./configure && CC=${CC} CXX=${CXX} ${MAKE}
cityhash: cityhash/cityhash/src/.libs/libcityhash.a

lz4/lz4/liblz4.a:
	cd lz4 && rm -rf lz4-*/ || true
	cd lz4 && tar -zxf lz4-1.7.5.tar.gz
	cd lz4/lz4 && CC=${CC} CXX=${CXX} ${MAKE}
lz4: lz4/lz4/liblz4.a

clickhouse-cpp/clickhouse-cpp/clickhouse/libclickhouse-cpp-lib.a:
	cd clickhouse-cpp && rm -rf clickhouse-cpp/ || true
	cd clickhouse-cpp && tar -zxf clickhouse-cpp.tar.gz
ifeq ($(CXX),clang++)
	cd clickhouse-cpp/clickhouse-cpp && sed -i 's/"-O2 -pthread -Wall -Wextra -Werror"/"-O2 -pthread -Wall -Wextra -Werror -Wno-deprecated-copy"/' CMakeLists.txt
endif
	cd clickhouse-cpp/clickhouse-cpp && cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo .
	cd clickhouse-cpp/clickhouse-cpp && patch clickhouse/columns/column.h < ../column.h.diff
	cd clickhouse-cpp/clickhouse-cpp && CC=${CC} CXX=${CXX} ${MAKE}

clickhouse-cpp: clickhouse-cpp/clickhouse-cpp/clickhouse/libclickhouse-cpp-lib.a


libdaemon/libdaemon/libdaemon/.libs/libdaemon.a:
	cd libdaemon && rm -rf libdaemon-*/ || true
	cd libdaemon && tar -zxf libdaemon-0.14.tar.gz
	cd libdaemon/libdaemon && cp ../config.guess . && chmod +x config.guess && ./configure --disable-examples
	cd libdaemon/libdaemon && CC=${CC} CXX=${CXX} ${MAKE}

libdaemon: libdaemon/libdaemon/libdaemon/.libs/libdaemon.a

jemalloc/jemalloc/lib/libjemalloc.a:
	cd jemalloc && rm -rf jemalloc-*/ || true
	cd jemalloc && tar --no-same-owner -jxf jemalloc-5.2.0.tar.bz2
	cd jemalloc/jemalloc && patch src/jemalloc.c < ../issue823.520.patch
	cd jemalloc/jemalloc && patch src/jemalloc.c < ../issue2358.patch
	cd jemalloc/jemalloc && ./configure ${MYJEOPT}
#	cd jemalloc/jemalloc && sed -i -e 's/-O3 /-O3 -fPIC /' Makefile
	cd jemalloc/jemalloc && CC=${CC} CXX=${CXX} ${MAKE}

jemalloc: jemalloc/jemalloc/lib/libjemalloc.a

WITHASAN := $(shell echo $(WITHASAN))

mariadb-client-library/mariadb_client/libmariadb/libmariadbclient.a: libssl/openssl/libssl.a
	cd mariadb-client-library && rm -rf mariadb-connector-c-*/ || true
	cd mariadb-client-library && tar -zxf mariadb-connector-c-3.1.9-src.tar.gz
	cd mariadb-client-library/mariadb_client && patch ./plugins/auth/CMakeLists.txt < ../plugin_auth_CMakeLists.txt.patch
	cd mariadb-client-library/mariadb_client && patch -p0 < ../ConnectorName.patch
	cd mariadb-client-library/mariadb_client && cmake . -DOPENSSL_ROOT_DIR=$(shell pwd)/libssl/openssl/ -DOPENSSL_LIBRARIES=$(shell pwd)/libssl/openssl/ .
#	cd mariadb-client-library/mariadb_client && cmake . -DOPENSSL_ROOT_DIR=/usr/local/opt/openssl # this is needed on MacOSX
ifeq ($(PROXYDEBUG),1)
	cd mariadb-client-library/mariadb_client && patch include/ma_context.h < ../ma_context.h.patch
else ifeq ($(USEVALGRIND),1)
	cd mariadb-client-library/mariadb_client && patch include/ma_context.h < ../ma_context.h.patch
endif
	cd mariadb-client-library/mariadb_client && patch libmariadb/mariadb_stmt.c < ../mariadb_stmt.c.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/mariadb_lib.c < ../mariadb_lib.c.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/mariadb_lib.c < ../mariadb_lib.c.collation.patch # make sure this path is applied after mariadb_lib.c.patch
#	cd mariadb-client-library/mariadb_client && patch libmariadb/net.c < ../net.c.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/mariadb_async.c < ../mariadb_async.c.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_password.c < ../ma_password.c.patch
#	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_secure.c < ../ma_secure.c.patch
	cd mariadb-client-library/mariadb_client && patch include/mysql.h < ../mysql.h.patch
	cd mariadb-client-library/mariadb_client && patch include/mariadb_com.h < ../mariadb_com.h.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_alloc.c < ../ma_alloc.c.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_charset.c < ../ma_charset.c.patch
#	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_pvio.c < ../ma_pvio.c.patch
	cd mariadb-client-library/mariadb_client && patch unittest/libmariadb/basic-t.c < ../unittest_basic-t.c.patch
	cd mariadb-client-library/mariadb_client && patch unittest/libmariadb/charset.c < ../unittest_charset.c.patch
	cd mariadb-client-library/mariadb_client && patch -p0 < ../client_deprecate_eof.patch
	cd mariadb-client-library/mariadb_client && patch -p0 < ../cr_new_stmt_metadata_removal.patch
	cd mariadb-client-library/mariadb_client && patch -p0 < ../ps_buffer_stmt_read_all_rows.patch
ifeq ($(WITHASAN),1)
	cd mariadb-client-library/mariadb_client && patch -p0 < ../mariadb_asan.patch
endif
	cd mariadb-client-library/mariadb_client && patch libmariadb/mariadb_dyncol.c < ../mariadb_dyncol.c-multiplication-overflow.patch
	cd mariadb-client-library/mariadb_client && patch libmariadb/ma_array.c < ../ma_array.c-multiplication-overflow.patch
	cd mariadb-client-library/mariadb_client && patch zlib/zutil.c < ../zutil.c-multiplication-overflow.patch
	cd mariadb-client-library/mariadb_client && CC=${CC} CXX=${CXX} ${MAKE} mariadbclient
# cd mariadb-client-library/mariadb_client/include && make my_config.h

mariadb_client: mariadb-client-library/mariadb_client/libmariadb/libmariadbclient.a


sqlite3/sqlite3/sqlite3.o:
	cd sqlite3 && rm -rf sqlite-amalgamation-*/ || true
	cd sqlite3 && tar -zxf sqlite-amalgamation-3190200.tar.gz
	cd sqlite3/sqlite3 && patch sqlite3.c < ../from_unixtime.patch
	cd sqlite3/sqlite3 && patch sqlite3.c < ../sqlite3.c-multiplication-overflow.patch
	cd sqlite3/sqlite3 && ${CC} ${MYCFLAGS} -fPIC -c -o sqlite3.o sqlite3.c -DSQLITE_ENABLE_MEMORY_MANAGEMENT -DSQLITE_ENABLE_JSON1 -DSQLITE_DLL=1
	cd sqlite3/sqlite3 && ${CC} -shared -o libsqlite3.so sqlite3.o

sqlite3: sqlite3/sqlite3/sqlite3.o

libconfig/libconfig/lib/.libs/libconfig++.a:
	cd libconfig && rm -rf libconfig-*/ || true
	cd libconfig && tar -zxf libconfig-1.7.2.tar.gz
	cd libconfig/libconfig && ./configure --disable-examples
	cd libconfig/libconfig && CC=${CC} CXX=${CXX} ${MAKE}

libconfig: libconfig/libconfig/lib/.libs/libconfig++.a

prometheus-cpp/prometheus-cpp/lib/libprometheus-cpp-core.a:
	cd prometheus-cpp && rm -rf prometheus-cpp-*/ || true
	cd prometheus-cpp && tar -zxf v0.9.0.tar.gz
	cd prometheus-cpp && tar --strip-components=1 -zxf civetweb-v1.11.tar.gz -C prometheus-cpp/3rdparty/civetweb
	cd prometheus-cpp/prometheus-cpp && patch -p1 < ../serial_exposer.patch
	cd prometheus-cpp/prometheus-cpp && patch -p0 < ../registry_counters_reset.patch
	cd prometheus-cpp/prometheus-cpp && patch -p0 < ../include_limits.patch
	cd prometheus-cpp/prometheus-cpp && cmake . -DBUILD_SHARED_LIBS=OFF -DENABLE_TESTING=OFF -DENABLE_PUSH=OFF
	cd prometheus-cpp/prometheus-cpp && CC=${CC} CXX=${CXX} ${MAKE}

prometheus-cpp: prometheus-cpp/prometheus-cpp/lib/libprometheus-cpp-core.a

re2/re2/obj/libre2.a:
	cd re2 && rm -rf re2-*/ || true
	cd re2 && tar -zxf re2.tar.gz
#	cd re2/re2 && sed -i -e 's/-O3 -g /-O3 -fPIC /' Makefile
#	cd re2/re2 && patch util/mutex.h < ../mutex.h.patch
	cd re2/re2 && patch re2/onepass.cc < ../onepass.cc-multiplication-overflow.patch
	cd re2/re2 && sed -i -e 's/-O3 /-O3 -fPIC -DMEMORY_SANITIZER -DRE2_ON_VALGRIND /' Makefile
	cd re2/re2 && sed -i -e 's/RE2_CXXFLAGS?=-std=c++11 /RE2_CXXFLAGS?=-std=c++11 -fPIC /' Makefile
	cd re2/re2 && CC=${CC} CXX=${CXX} ${MAKE}

re2: re2/re2/obj/libre2.a

pcre/pcre/.libs/libpcre.a:
	cd pcre && rm -rf pcre-*/ || true
	cd pcre && tar -zxf pcre-8.44.tar.gz
	cd pcre/pcre && patch pcretest.c < ../pcretest.c-multiplication-overflow.patch
	cd pcre/pcre && ./configure
	cd pcre/pcre && CC=${CC} CXX=${CXX} ${MAKE}
pcre: pcre/pcre/.libs/libpcre.a

cleanpart:
	cd mariadb-client-library && rm -rf mariadb-connector-c-*/ || true
	cd jemalloc && rm -rf jemalloc-*/ || true
	cd sqlite3 && rm -rf sqlite-amalgamation-*/ || true
.PHONY: cleanpart

cleanall:
	cd libinjection && rm -rf libinjection-*/ || true
	cd libhttpserver && rm -rf libhttpserver-*/ || true
	cd libdaemon && rm -rf libdaemon-*/ || true
	cd jemalloc && rm -rf jemalloc-*/ || true
	cd mariadb-client-library && rm -rf mariadb-connector-c-*/ || true
	cd libconfig && rm -rf libconfig-*/ || true
	cd re2 && rm -rf re2-*/ || true
	cd pcre && rm -rf pcre-*/ || true
	cd sqlite3 && rm -rf sqlite-amalgamation-*/ || true
	cd clickhouse-cpp/ && rm -rf clickhouse-cpp/ || true
	cd lz4 && rm -rf lz4-*/ || true
	cd libmicrohttpd && rm -rf libmicrohttpd-*/ || true
	cd libmicrohttpd && rm -f libmicrohttpd || true
	cd curl && rm -rf curl-*/ || true
	cd libev && rm -rf libev-*/ || true
	cd libssl && rm -rf openssl-openssl-*/ || true
	cd libconfig && rm -rf libconfig-*/ || true
	cd prometheus-cpp && rm -rf prometheus-cpp-*/ || true
	cd cityhash && rm -rf cityhash/ || true
.PHONY: cleanall

