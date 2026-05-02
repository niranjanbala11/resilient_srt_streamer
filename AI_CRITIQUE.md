AI generated code:
=========================
#include <gst/gst.h>
#include <glib.h>
#include <iostream>

GstElement *pipeline, *selector, *srtsink;
GstPad *file_pad = NULL;
GstPad *fallback_pad = NULL;

// ------------------------
// Stats Loop (every 1 sec)
// ------------------------
gboolean print_srt_stats(gpointer data) {
    GstStructure *stats = NULL;

    g_object_get(srtsink, "stats", &stats, NULL);

    if (stats) {
        guint64 rtt = 0;
        guint64 pkt_loss = 0;

        gst_structure_get_uint64(stats, "rtt", &rtt);
        gst_structure_get_uint64(stats, "pkt-loss", &pkt_loss);

        std::cout << "[SRT STATS] RTT: " << rtt
                  << " ms | Packet Loss: " << pkt_loss << std::endl;

        gst_structure_free(stats);
    }

    return TRUE; // keep looping
}

// ------------------------
// Switch to fallback
// ------------------------
void switch_to_fallback() {
    std::cout << "⚠️ Switching to fallback stream...\n";

    if (fallback_pad) {
        g_object_set(selector, "active-pad", fallback_pad, NULL);
    }
}

// ------------------------
// Bus Callback
// ------------------------
gboolean bus_call(GstBus *bus, GstMessage *msg, gpointer data) {
    switch (GST_MESSAGE_TYPE(msg)) {

        case GST_MESSAGE_EOS:
            std::cout << "End of stream detected\n";
            switch_to_fallback();
            break;

        case GST_MESSAGE_ERROR: {
            GError *err;
            gchar *debug;

            gst_message_parse_error(msg, &err, &debug);
            std::cerr << "Error: " << err->message << std::endl;

            g_error_free(err);
            g_free(debug);

            switch_to_fallback();
            break;
        }

        default:
            break;
    }

    return TRUE;
}

// ------------------------
// Decodebin pad-added
// ------------------------
void on_pad_added(GstElement *src, GstPad *new_pad, gpointer data) {
    GstPad *sink_pad = gst_element_get_static_pad((GstElement *)data, "sink");

    if (gst_pad_is_linked(sink_pad)) {
        gst_object_unref(sink_pad);
        return;
    }

    if (gst_pad_link(new_pad, sink_pad) != GST_PAD_LINK_OK) {
        std::cerr << "Failed to link decodebin\n";
    } else {
        std::cout << "Decodebin linked successfully\n";
    }

    gst_object_unref(sink_pad);
}

// ------------------------
// Main
// ------------------------
int main(int argc, char *argv[]) {

    gst_init(&argc, &argv);

    if (argc < 2) {
        std::cout << "Usage: " << argv[0] << " <video file>\n";
        return -1;
    }

    GMainLoop *loop = g_main_loop_new(NULL, FALSE);

    // ------------------------
    // Elements
    // ------------------------
    GstElement *filesrc = gst_element_factory_make("filesrc", "file-source");
    GstElement *decodebin = gst_element_factory_make("decodebin", "decoder");

    GstElement *queue1 = gst_element_factory_make("queue", "queue1");

    GstElement *videotestsrc = gst_element_factory_make("videotestsrc", "fallback-src");
    GstElement *queue2 = gst_element_factory_make("queue", "queue2");

    selector = gst_element_factory_make("input-selector", "selector");

    GstElement *x264enc = gst_element_factory_make("x264enc", "encoder");
    GstElement *mpegtsmux = gst_element_factory_make("mpegtsmux", "mux");

    srtsink = gst_element_factory_make("srtsink", "srt-output");

    pipeline = gst_pipeline_new("srt-streamer");

    if (!pipeline || !filesrc || !decodebin || !queue1 ||
        !videotestsrc || !queue2 || !selector ||
        !x264enc || !mpegtsmux || !srtsink) {

        std::cerr << "Failed to create elements\n";
        return -1;
    }

    // ------------------------
    // Configure elements
    // ------------------------
    g_object_set(filesrc, "location", argv[1], NULL);

    g_object_set(videotestsrc, "is-live", TRUE, NULL);

    g_object_set(x264enc,
                 "tune", 0x00000004, // zerolatency
                 "speed-preset", 1,  // ultrafast
                 "bitrate", 1000,
                 "key-int-max", 30,
                 NULL);

    g_object_set(srtsink,
                 "uri", "srt://127.0.0.1:9000",
                 "wait-for-connection", TRUE,
                 "latency", 50,
                 NULL);

    // ------------------------
    // Build pipeline
    // ------------------------
    gst_bin_add_many(GST_BIN(pipeline),
                     filesrc, decodebin, queue1,
                     videotestsrc, queue2,
                     selector, x264enc, mpegtsmux, srtsink,
                     NULL);

    gst_element_link(filesrc, decodebin);

    g_signal_connect(decodebin, "pad-added", G_CALLBACK(on_pad_added), queue1);

    gst_element_link(queue1, selector);
    gst_element_link(videotestsrc, queue2);
    gst_element_link(queue2, selector);

    gst_element_link(selector, x264enc);
    gst_element_link(x264enc, mpegtsmux);
    gst_element_link(mpegtsmux, srtsink);

    // ------------------------
    // Get selector pads
    // ------------------------
    file_pad = gst_element_get_request_pad(selector, "sink_%u");
    GstPad *queue1_src = gst_element_get_static_pad(queue1, "src");
    gst_pad_link(queue1_src, file_pad);
    gst_object_unref(queue1_src);

    fallback_pad = gst_element_get_request_pad(selector, "sink_%u");
    GstPad *queue2_src = gst_element_get_static_pad(queue2, "src");
    gst_pad_link(queue2_src, fallback_pad);
    gst_object_unref(queue2_src);

    // Start with file
    g_object_set(selector, "active-pad", file_pad, NULL);

    // ------------------------
    // Bus
    // ------------------------
    GstBus *bus = gst_pipeline_get_bus(GST_PIPELINE(pipeline));
    gst_bus_add_watch(bus, bus_call, loop);
    gst_object_unref(bus);

    // ------------------------
    // Stats timer
    // ------------------------
    g_timeout_add_seconds(1, print_srt_stats, NULL);

    // ------------------------
    // Run
    // ------------------------
    gst_element_set_state(pipeline, GST_STATE_PLAYING);
    std::cout << "Streaming started...\n";

    g_main_loop_run(loop);

    // Cleanup
    gst_element_set_state(pipeline, GST_STATE_NULL);
    gst_object_unref(pipeline);

    return 0;
}

Issues in AI-Generated Code:
=============================
1)Missing Format Negotiation (videoconvert / caps)
    Problem:
        decodebin outputs dynamic raw formats (e.g., NV12, RGB)
        x264enc expects I420
    Impact:
        Pipeline produced invalid or undecodable streams
        Receiver failed with:
            Can't typefind stream
            Stream doesn't contain enough data
    Solution:
        Added Proper Format Handling
        Introduced videoconvert to normalize formats
        Enforced I420 using capsfilter

2)Incorrect Linking
    Problem:
        Elements were linked multiple times
    Impact:
        Pipeline failed silently or never linked correctly
    Solution:
        Implemented Dynamic Pad Linking

3)No Buffering (Queues)
    Problem:
        No queue elements in pipeline
    Impact:
        Timing issues
        Frame drops
        Pipeline instability
        SRT socket errors during transitions
    Solution:
        Introduced queues between various elements

4)Rigid Encoder Configuration
    Problem:
        Hardcoded x264 parameters
    Impact:
        No flexibility for:
        different network conditions
        bitrate tuning
    Solution:
        Enabled Configurable Encoder Parameters
            Added CLI support for:
                bitrate
                preset
                GOP size

5)Incorrect SRT stats
    Problem:
        Collected SRT stats with crong parameter names
    Impact:
        The stats simply showed zero without giving actual values
    Solution:
        Collected proper SRT stats and calculated necessary stats using available stats