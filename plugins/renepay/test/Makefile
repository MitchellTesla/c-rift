# Note that these actually #include everything they need, except ccan/ and bitcoin/.
# That allows for unit testing of statics, and special effects.
PLUGIN_RENEPAY_TEST_SRC := $(wildcard plugins/renepay/test/run-*.c)
PLUGIN_RENEPAY_TEST_OBJS := $(PLUGIN_RENEPAY_TEST_SRC:.c=.o)
PLUGIN_RENEPAY_TEST_PROGRAMS := $(PLUGIN_RENEPAY_TEST_OBJS:.o=)

ALL_C_SOURCES += $(PLUGIN_RENEPAY_TEST_SRC)
ALL_TEST_PROGRAMS += $(PLUGIN_RENEPAY_TEST_PROGRAMS)
$(PLUGIN_RENEPAY_TEST_OBJS): $(PLUGIN_RENEPAY_SRC)

PLUGIN_RENEPAY_TEST_COMMON_OBJS :=		\
	plugins/renepay/dijkstra.o		\
	plugins/renepay/debug.o

$(PLUGIN_RENEPAY_TEST_PROGRAMS): $(PLUGIN_RENEPAY_TEST_COMMON_OBJS) $(PLUGIN_LIB_OBJS) $(PLUGIN_COMMON_OBJS) $(JSMN_OBJS) $(CCAN_OBJS) bitcoin/chainparams.o common/gossmap.o common/fp16.o common/dijkstra.o common/bolt12.o common/bolt12_merkle.o wire/bolt12$(EXP)_wiregen.o

check-renepay: $(PLUGIN_RENEPAY_TEST_PROGRAMS:%=unittest/%)

check-units: check-renepay
