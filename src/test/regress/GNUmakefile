#-------------------------------------------------------------------------
#
# GNUmakefile--
#    Makefile for src/test/regress (the regression tests)
#
# Portions Copyright (c) 1996-2011, PostgreSQL Global Development Group
# Portions Copyright (c) 1994, Regents of the University of California
#
# src/test/regress/GNUmakefile
#
#-------------------------------------------------------------------------

subdir = src/test/regress
top_builddir = ../../..
include $(top_builddir)/src/Makefile.global

# file with extra config for temp build
TEMP_CONF =
ifdef TEMP_CONFIG
TEMP_CONF += --temp-config=$(TEMP_CONFIG)
endif

# where to find psql for testing an existing installation
PSQLDIR = $(bindir)

# maximum simultaneous connections for parallel tests
MAXCONNOPT =
ifdef MAX_CONNECTIONS
MAXCONNOPT += --max-connections=$(MAX_CONNECTIONS)
endif

# locale
NOLOCALE =
ifdef NO_LOCALE
NOLOCALE += --no-locale
endif

# stuff to pass into build of pg_regress
EXTRADEFS = '-DHOST_TUPLE="$(host_tuple)"' \
	'-DMAKEPROG="$(MAKE)"' \
	'-DSHELLPROG="$(SHELL)"' \
	'-DDLSUFFIX="$(DLSUFFIX)"'

##
## Prepare for tests
##

# Build regression test driver

all: pg_regress$(X)

pg_regress$(X): pg_regress.o pg_regress_main.o | submake-libpgport
	$(CC) $(CFLAGS) $^ $(LDFLAGS) $(LDFLAGS_EX) $(LIBS) -o $@

# dependencies ensure that path changes propagate
pg_regress.o: pg_regress.c $(top_builddir)/src/port/pg_config_paths.h
	$(CC) $(CFLAGS) $(CPPFLAGS) -I$(top_builddir)/src/port $(EXTRADEFS) -c -o $@ $<

$(top_builddir)/src/port/pg_config_paths.h: $(top_builddir)/src/Makefile.global
	$(MAKE) -C $(top_builddir)/src/port pg_config_paths.h

install: all installdirs
	$(INSTALL_PROGRAM) pg_regress$(X) '$(DESTDIR)$(pgxsdir)/$(subdir)/pg_regress$(X)'

installdirs:
	$(MKDIR_P) '$(DESTDIR)$(pgxsdir)/$(subdir)'

uninstall:
	rm -f '$(DESTDIR)$(pgxsdir)/$(subdir)/pg_regress$(X)'


# Build dynamically-loaded object file for CREATE FUNCTION ... LANGUAGE C.

NAME = regress
OBJS = regress.o

include $(top_srcdir)/src/Makefile.shlib

all: all-lib

# Test input and expected files.  These are created by pg_regress itself, so we
# don't have a rule to create them.  We do need rules to clean them however.
input_files = $(patsubst $(srcdir)/input/%.source,sql/%.sql, $(wildcard $(srcdir)/input/*.source))
output_files := $(patsubst $(srcdir)/output/%.source,expected/%.out, $(wildcard $(srcdir)/output/*.source))


# not installed by default

regress_data_files = \
	$(filter-out $(addprefix $(srcdir)/,$(output_files)),$(wildcard $(srcdir)/expected/*.out)) \
	$(wildcard $(srcdir)/input/*.source) \
	$(wildcard $(srcdir)/output/*.source) \
	$(filter-out $(addprefix $(srcdir)/,$(input_files)),$(wildcard $(srcdir)/sql/*.sql)) \
	$(wildcard $(srcdir)/data/*.data) \
	$(srcdir)/parallel_schedule $(srcdir)/serial_schedule $(srcdir)/resultmap

install-tests: all install install-lib installdirs-tests
	$(MAKE) -C $(top_builddir)/contrib/spi install
	for file in $(regress_data_files); do \
	  $(INSTALL_DATA) $$file '$(DESTDIR)$(pkglibdir)/regress/'$$file || exit; \
	done

installdirs-tests: installdirs
	$(MKDIR_P)  $(patsubst $(srcdir)/%/,'$(DESTDIR)$(pkglibdir)/regress/%',$(sort $(dir $(regress_data_files))))


# Get some extra C modules from contrib/spi and contrib/dummy_seclabel...

all: refint$(DLSUFFIX) autoinc$(DLSUFFIX) dummy_seclabel$(DLSUFFIX)

refint$(DLSUFFIX): $(top_builddir)/contrib/spi/refint$(DLSUFFIX)
	cp $< $@

autoinc$(DLSUFFIX): $(top_builddir)/contrib/spi/autoinc$(DLSUFFIX)
	cp $< $@

dummy_seclabel$(DLSUFFIX): $(top_builddir)/contrib/dummy_seclabel/dummy_seclabel$(DLSUFFIX)
	cp $< $@

$(top_builddir)/contrib/spi/refint$(DLSUFFIX): $(top_srcdir)/contrib/spi/refint.c
	$(MAKE) -C $(top_builddir)/contrib/spi refint$(DLSUFFIX)

$(top_builddir)/contrib/spi/autoinc$(DLSUFFIX): $(top_srcdir)/contrib/spi/autoinc.c
	$(MAKE) -C $(top_builddir)/contrib/spi autoinc$(DLSUFFIX)

$(top_builddir)/contrib/dummy_seclabel/dummy_seclabel$(DLSUFFIX): $(top_builddir)/contrib/dummy_seclabel/dummy_seclabel.c
	$(MAKE) -C $(top_builddir)/contrib/dummy_seclabel dummy_seclabel$(DLSUFFIX)

# Tablespace setup

.PHONY: tablespace-setup
tablespace-setup:
	rm -rf ./testtablespace
	mkdir ./testtablespace


##
## Run tests
##

pg_regress_call = ./pg_regress --inputdir=$(srcdir) --dlpath=. $(if $(MULTIBYTE),--multibyte=$(MULTIBYTE)) $(NOLOCALE)

check: all tablespace-setup
	$(pg_regress_call) --temp-install=./tmp_check --top-builddir=$(top_builddir) --schedule=$(srcdir)/parallel_schedule $(MAXCONNOPT) $(TEMP_CONF) $(EXTRA_TESTS)

installcheck: all tablespace-setup
	$(pg_regress_call) --psqldir=$(PSQLDIR) --schedule=$(srcdir)/serial_schedule $(EXTRA_TESTS)

installcheck-parallel: all tablespace-setup
	$(pg_regress_call) --psqldir=$(PSQLDIR) --schedule=$(srcdir)/parallel_schedule $(MAXCONNOPT) $(EXTRA_TESTS)

standbycheck: all
	$(pg_regress_call) --psqldir=$(PSQLDIR) --schedule=$(srcdir)/standby_schedule --use-existing

# old interfaces follow...

runcheck: check
runtest: installcheck
runtest-parallel: installcheck-parallel

bigtest: all tablespace-setup
	$(pg_regress_call) --psqldir=$(PSQLDIR) --schedule=$(srcdir)/serial_schedule numeric_big

bigcheck: all tablespace-setup
	$(pg_regress_call) --temp-install=./tmp_check --top-builddir=$(top_builddir) --schedule=$(srcdir)/parallel_schedule $(MAXCONNOPT) numeric_big


##
## Clean up
##

clean distclean maintainer-clean: clean-lib
# things built by `all' target
	rm -f $(OBJS) refint$(DLSUFFIX) autoinc$(DLSUFFIX) dummy_seclabel$(DLSUFFIX)
	rm -f pg_regress_main.o pg_regress.o pg_regress$(X)
# things created by various check targets
	rm -f $(output_files) $(input_files)
	rm -rf testtablespace
	rm -rf results tmp_check log
	rm -f regression.diffs regression.out regress.out run_check.out
