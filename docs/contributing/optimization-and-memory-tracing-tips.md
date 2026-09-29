You can output a Flight Recording of a full test run using:

```
mvn verify --% -DargLine.extra="-XX:StartFlightRecording=filename=target/allocations.jfr,settings=profile"
```

Note that was used on a Windows machine you may need to adjust for other OS.

You can get a Flight Recording from a running process. To start:

```
jcmd {PID} JFR.start name=SteadyState settings=profile filename={ABSOLUTE_FILE_PATH_FOR.JFR}
```

and to stop:
```
jcmd {PID} JFR.stop name=SteadyState 
```

To generate traffic on a local server for load testing, you can use something like this (you may want to adjust for specific datasets and/or output formats).

``` py
import random
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timedelta
import requests
from requests.adapters import HTTPAdapter
from urllib3.util import Retry

# ==============================================================================
# CONFIGURATION
# ==============================================================================
BASE_URL = "http://localhost:8080/erddap"

CONCURRENT_USERS = 10        # Parallel worker threads
DURATION_SECONDS = 300        # Test duration in seconds
REQUEST_TIMEOUT_SEC = 30     # Request timeout threshold

# ==============================================================================
# VALID DATASET QUERY GENERATORS (DEFINED FROM datasets.xml)
# ==============================================================================

def get_random_format():
    """Randomly select a valid format for the query."""
    return random.choice(["csv", "json", "dods", "ncHeader", "htmlTable", "nc", "parquet", "croissant"])

def query_test_global():
    """CalCOFI Subsurface Physical Data (Table)."""
    fmt = get_random_format()
    # Latitude: 20 to 50, Longitude: 200 to 250 (degrees East)
    lat1 = round(random.uniform(20.0, 35.0), 2)
    lat2 = round(lat1 + random.uniform(10.0, 20.0), 2)
    lon1 = round(random.uniform(-164.0, -130.0), 2)
    lon2 = round(lon1 + random.uniform(10.0, 20.0), 2)
    
    url = (
        f"{BASE_URL}/tabledap/testGlobal.{fmt}?"
        f"line_station,longitude,latitude,time,depth,temperature,salinity"
        f"&latitude>={lat1}&latitude<={lat2}"
        f"&longitude>={lon1}&longitude<={lon2}"
    )
    return url

def query_mini_ndbc():
    """NDBC Standard Meteorological Buoy Data (Table)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/tabledap/miniNdbc.{fmt}?"
        f"station,longitude,latitude,time,wspd,atmp,wtmp,bar"
    )
    return url

def query_pmel_tao_airt():
    """TAO/TRITON Daily Air Temperature Data (Table)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/tabledap/pmelTaoDyAirt.{fmt}?"
        f"array,station,longitude,latitude,time,depth,AT_21"
    )
    return url

def query_globec_bottle():
    """GLOBEC NEP Rosette Bottle Data (Table)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/tabledap/testGlobecBottle.{fmt}?"
        f"cruise_id,ship,cast,longitude,latitude,time,sal00,temperature0"
    )
    return url

def query_epaseamap():
    """EPA SeaMap Water Station Profiles (Table)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/tabledap/epaseamapTimeSeriesProfiles.{fmt}?"
        f"station_name,station,latitude,longitude,time,depth,WaterTemperature,salinity"
    )
    return url

def query_ca_market_catch():
    """California Fish Market Catch Monthly (Table)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/tabledap/erdCAMarCatSM.{fmt}?"
        f"time,year,fish,port,landings"
    )
    return url

def query_time_axis():
    """Historical Total Solar Irradiance (Table)."""
    fmt = get_random_format()
    url = f"{BASE_URL}/tabledap/testTimeAxis.{fmt}?time,irradiance"
    return url

def query_zarr_compressed_grid():
    """Zarr Compressed Grid (Grid)."""
    fmt = get_random_format()
    url = (
        f"{BASE_URL}/griddap/zarr_gridCompressedData.{fmt}?"
        f"compressed_deflate1[0:1:1][0:1:1]"
    )
    return url

def query_table_metadata_and_info():
    """Catalog and Dataset Info Page Queries."""
    dataset_id = random.choice([
        "testGlobal", "testGlobecBottle" , "pmelTaoDyAirt", 
        "erdCAMarCatSM", "miniNdbc", "epaseamapTimeSeriesProfiles"
    ])
    endpoint = random.choice([
        f"/info/{dataset_id}/index.html",
        f"/tabledap/{dataset_id}.das",
        f"/tabledap/{dataset_id}.dds",
        "/info/index.html"
    ])
    return f"{BASE_URL}{endpoint}"

def query_grid_metadata_and_info():
    """Catalog and Dataset Info Page Queries."""
    dataset_id = random.choice([
        "testGridWav", "nceiPH53sstd1day",
        "erdMH1chla1day", "zarr_gridCompressedData"
    ])
    endpoint = random.choice([
        f"/info/{dataset_id}/index.html",
        f"/griddap/{dataset_id}.das",
        "/info/index.html"
    ])
    return f"{BASE_URL}{endpoint}"

# ==============================================================================
# WEIGHTED QUERY SELECTOR
# ==============================================================================
def generate_valid_url():
    """Weighted random selection across valid XML datasets."""
    generators = [
        (query_test_global, 0.25),
        (query_mini_ndbc, 0.15),
        (query_epaseamap, 0.15),
        # (query_hawaii_soda_grid, 0.15),
        (query_pmel_tao_airt, 0.10),
        (query_ca_market_catch, 0.05),
        (query_globec_bottle, 0.05),
        (query_zarr_compressed_grid, 0.05),
        (query_table_metadata_and_info, 0.05),
        (query_grid_metadata_and_info, 0.05)
    ]
    
    roll = random.random()
    cumulative = 0.0
    for gen, weight in generators:
        cumulative += weight
        if roll <= cumulative:
            return gen()
    return query_test_global()

# ==============================================================================
# WORKER LOOP & MONITORING
# ==============================================================================
def worker_loop(worker_id, stop_time, stats):
    session = requests.Session()
    
    # Configure an HTTPAdapter with explicit pool sizes and Keep-Alive
    adapter = HTTPAdapter(
        pool_connections=20,  # Max cached hosts
        pool_maxsize=20,      # Max cached sockets per host
        max_retries=Retry(total=3, backoff_factor=0.1)
    )
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    # Force HTTP Keep-Alive header
    session.headers.update({"Connection": "keep-alive"})

    while time.time() < stop_time:
        url = generate_valid_url()
        start = time.time()
        try:
            response = session.get(url, timeout=REQUEST_TIMEOUT_SEC)
            elapsed = time.time() - start
            status = response.status_code
            
            # Consume stream content fully to release the socket back to the pool
            _ = response.content
            
            stats['requests'] += 1
            if status == 200:
                stats['success'] += 1
            else:
                stats['errors'] += 1
                stats['status_codes'][status] = stats['status_codes'].get(status, 0) + 1
                
        except Exception as e:
            stats['errors'] += 1
            stats['exceptions'] += 1

# ==============================================================================
# MAIN EXECUTION
# ==============================================================================
if __name__ == "__main__":
    print(f"Starting Tailored ERDDAP Load Test...")
    print(f"Target Server: {BASE_URL}")
    print(f"Threads: {CONCURRENT_USERS} | Duration: {DURATION_SECONDS} seconds\n")

    stop_time = time.time() + DURATION_SECONDS
    stats = {
        'requests': 0, 
        'success': 0, 
        'errors': 0, 
        'exceptions': 0,
        'status_codes': {}
    }

    start_time = time.time()
    
    with ThreadPoolExecutor(max_workers=CONCURRENT_USERS) as executor:
        futures = [
            executor.submit(worker_loop, i, stop_time, stats)
            for i in range(CONCURRENT_USERS)
        ]
        for future in futures:
            future.result()

    total_time = time.time() - start_time
    rps = stats['requests'] / total_time if total_time > 0 else 0

    print("=" * 50)
    print("LOAD TEST RESULTS")
    print("=" * 50)
    print(f"Total Duration : {total_time:.2f} seconds")
    print(f"Total Requests : {stats['requests']}")
    print(f"Successful 200 : {stats['success']}")
    print(f"Failed Non-200 : {stats['errors']}")
    print(f"Exceptions     : {stats['exceptions']}")
    if stats['status_codes']:
        print(f"Error Statuses : {stats['status_codes']}")
    print(f"Throughput     : {rps:.2f} req/sec")
    print("=" * 50)
    
```

I used some scripts to read the jfr files.

First up parsing the file and outputing sources of major memory allocations:

``` java
import jdk.jfr.consumer.*;
import java.io.PrintWriter;
import java.nio.file.Path;
import java.util.*;

public class ParseJFR {
    private static final Set<String> ALLOC_EVENTS = Set.of(
        "jdk.ObjectAllocationSample",
        "jdk.ObjectAllocationInNewTLAB",
        "jdk.ObjectAllocationOutsideTLAB"
    );

    public static void main(String[] args) throws Exception {
        String jfrPath = args.length > 0 ? args[0] : "opt_pass_2.jfr";
        String foldedOutputFile = args.length > 1 ? args[1] : "allocations.folded";
        int maxFrames = 3;

        Map<String, Long> allocations = new HashMap<>();
        Map<String, Long> classAllocations = new HashMap<>();
        Map<String, Long> collapsedStacks = new HashMap<>();
        long totalEventsHandled = 0;

        try (RecordingFile rf = new RecordingFile(Path.of(jfrPath))) {
            while (rf.hasMoreEvents()) {
                RecordedEvent event = rf.readEvent();
                if (!ALLOC_EVENTS.contains(event.getEventType().getName())) continue;

                totalEventsHandled++;
                long weight = getEventWeight(event);

                RecordedClass rc = event.hasField("objectClass") ? event.getClass("objectClass") : null;
                String allocatedType = (rc != null) ? rc.getName() : "Unknown";

                // 1. Class-only aggregation (ignoring stack traces)
                classAllocations.merge(allocatedType, weight, Long::sum);

                RecordedStackTrace st = event.getStackTrace();
                if (st != null && !st.getFrames().isEmpty()) {
                    List<RecordedFrame> frames = st.getFrames();
                    List<String> appFrames = new ArrayList<>();

                    // Extract app-specific frames for filtered call chain
                    for (RecordedFrame f : frames) {
                        RecordedMethod m = f.getMethod();
                        if (m == null || m.getType() == null) continue;
                        String className = m.getType().getName();
                        if (className.startsWith("com.cohort") || className.startsWith("gov.noaa")) {
                            appFrames.add(className + "." + m.getName() + ":" + f.getLineNumber());
                        }
                    }

                    if (appFrames.isEmpty()) {
                        for (RecordedFrame f : frames) {
                            RecordedMethod m = f.getMethod();
                            if (m != null && m.getType() != null) {
                                appFrames.add(m.getType().getName() + "." + m.getName() + ":" + f.getLineNumber());
                            }
                        }
                    }

                    // 2. Filtered Call Chain Aggregation (Leaf <-- Caller)
                    int depth = Math.min(maxFrames, appFrames.size());
                    String callChain = String.join(" <-- ", appFrames.subList(0, depth));
                    String key = String.format("%-30s | %s", allocatedType, callChain);
                    allocations.merge(key, weight, Long::sum);

                    // 3. Collapsed Stack Export (Root -> Leaf -> [AllocatedType])
                    List<String> rawStackReversed = new ArrayList<>();
                    for (int i = frames.size() - 1; i >= 0; i--) {
                        RecordedFrame f = frames.get(i);
                        RecordedMethod m = f.getMethod();
                        if (m != null && m.getType() != null) {
                            rawStackReversed.add(m.getType().getName() + "." + m.getName());
                        }
                    }
                    rawStackReversed.add("[" + allocatedType + "]");
                    String collapsedKey = String.join(";", rawStackReversed);
                    collapsedStacks.merge(collapsedKey, weight, Long::sum);
                }
            }
        }

        // Export collapsed flamegraph file
        try (PrintWriter pw = new PrintWriter(foldedOutputFile)) {
            collapsedStacks.forEach((stack, bytes) -> pw.println(stack + " " + bytes));
        }

        System.out.printf("Processed %,d allocation events.%n", totalEventsHandled);
        System.out.println("Exported collapsed flamegraph stack to: " + foldedOutputFile + "\n");

        // Output Class-Only Allocations
        System.out.println("=== TOP ALLOCATED CLASSES (GLOBAL) ===");
        System.out.printf("%-15s | %s%n", "TOTAL BYTES", "CLASS NAME");
        System.out.println("-".repeat(60));
        classAllocations.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(15)
            .forEach(e -> System.out.printf("%-15s | %s%n", formatBytes(e.getValue()), e.getKey()));

        System.out.println("\n=== TOP ALLOCATION CALL CHAINS ===");
        System.out.printf("%-15s | %-30s | %s%n", "TOTAL BYTES", "ALLOCATED TYPE", "CALL CHAIN");
        System.out.println("-".repeat(110));
        allocations.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(20)
            .forEach(e -> System.out.printf("%-15s | %s%n", formatBytes(e.getValue()), e.getKey()));
    }

    private static long getEventWeight(RecordedEvent event) {
        if (event.hasField("weight")) return event.getLong("weight");
        if (event.hasField("allocationSize")) return event.getLong("allocationSize");
        if (event.hasField("tlabSize")) return event.getLong("tlabSize");
        return 0L;
    }

    private static String formatBytes(long bytes) {
        if (bytes < 1024) return bytes + " B";
        int exp = (int) (Math.log(bytes) / Math.log(1024));
        return String.format("%.2f %cB", bytes / Math.pow(1024, exp), "KMGTPE".charAt(exp - 1));
    }
}

```

and then comparing two jfr files, so you could have a baseline and then want to see if optimzationz improved it:

``` java
import jdk.jfr.consumer.*;
import java.nio.file.Path;
import java.util.*;

public class CompareJFR {
    private static final Set<String> ALLOC_EVENTS = Set.of(
        "jdk.ObjectAllocationSample",
        "jdk.ObjectAllocationInNewTLAB",
        "jdk.ObjectAllocationOutsideTLAB"
    );

    public static void main(String[] args) throws Exception {
        if (args.length < 2) {
            System.out.println("Usage: java CompareJFR <baseline.jfr> <optimized.jfr> [maxFrames]");
            return;
        }

        Path baselinePath = Path.of(args[0]);
        Path optimizedPath = Path.of(args[1]);
        int maxFrames = args.length > 2 ? Integer.parseInt(args[2]) : 3;

        System.out.println("Parsing baseline:  " + baselinePath);
        JfrData baselineData = parseJfr(baselinePath, maxFrames);

        System.out.println("Parsing optimized: " + optimizedPath);
        JfrData optimizedData = parseJfr(optimizedPath, maxFrames);

        // Print Global Overview
        System.out.println("\n====================================================================================================");
        System.out.println("OVERALL ALLOCATION SUMMARY");
        System.out.printf("Baseline Total : %-15s%n", formatBytes(baselineData.totalBytes));
        System.out.printf("Optimized Total: %-15s%n", formatBytes(optimizedData.totalBytes));
        long netDelta = optimizedData.totalBytes - baselineData.totalBytes;
        System.out.printf("Net Change     : %-15s (%+.2f%%)%n", formatBytes(netDelta), calculatePct(baselineData.totalBytes, optimizedData.totalBytes));
        System.out.println("====================================================================================================");

        // 1. Class-Only Diff (100% Immune to code refactoring and line shifts)
        printDiffTable("CLASS-LEVEL ALLOCATION DIFF", 
            buildDiffs(baselineData.classAllocations, optimizedData.classAllocations), 10);

        // 2. Line-Agnostic Call Chain Diffs
        List<DiffEntry> chainDiffs = buildDiffs(baselineData.chainAllocations, optimizedData.chainAllocations);

        System.out.println("\n=== TOP ALLOCATION SAVINGS (METHOD CHAINS) ===");
        printHeader();
        chainDiffs.stream()
            .filter(d -> d.delta < 0)
            .sorted(Comparator.comparingLong(d -> d.delta))
            .limit(15)
            .forEach(DiffEntry::printRow);

        System.out.println("\n=== TOP REGRESSIONS & INCREASES (METHOD CHAINS) ===");
        printHeader();
        chainDiffs.stream()
            .filter(d -> d.delta > 0)
            .sorted((a, b) -> Long.compare(b.delta, a.delta))
            .limit(15)
            .forEach(DiffEntry::printRow);
    }

    private static JfrData parseJfr(Path path, int maxFrames) throws Exception {
        Map<String, Long> chainAllocations = new HashMap<>();
        Map<String, Long> classAllocations = new HashMap<>();
        long totalBytes = 0;

        try (RecordingFile rf = new RecordingFile(path)) {
            while (rf.hasMoreEvents()) {
                RecordedEvent event = rf.readEvent();
                if (!ALLOC_EVENTS.contains(event.getEventType().getName())) continue;

                long weight = getEventWeight(event);
                totalBytes += weight;

                RecordedClass rc = event.hasField("objectClass") ? event.getClass("objectClass") : null;
                String allocatedType = (rc != null) ? rc.getName() : "Unknown";

                classAllocations.merge(allocatedType, weight, Long::sum);

                RecordedStackTrace st = event.getStackTrace();
                if (st != null && !st.getFrames().isEmpty()) {
                    List<RecordedFrame> frames = st.getFrames();
                    List<String> appFrames = new ArrayList<>();

                    // Collect method signatures strictly WITHOUT line numbers
                    for (RecordedFrame f : frames) {
                        RecordedMethod m = f.getMethod();
                        if (m == null || m.getType() == null) continue;
                        String className = m.getType().getName();
                        if (className.startsWith("com.cohort") || className.startsWith("gov.noaa")) {
                            appFrames.add(className + "." + m.getName());
                        }
                    }

                    if (appFrames.isEmpty()) {
                        for (RecordedFrame f : frames) {
                            RecordedMethod m = f.getMethod();
                            if (m != null && m.getType() != null) {
                                appFrames.add(m.getType().getName() + "." + m.getName());
                            }
                        }
                    }

                    int depth = Math.min(maxFrames, appFrames.size());
                    String callChain = String.join(" <-- ", appFrames.subList(0, depth));
                    String key = String.format("%s | %s", allocatedType, callChain);

                    chainAllocations.merge(key, weight, Long::sum);
                }
            }
        }
        return new JfrData(chainAllocations, classAllocations, totalBytes);
    }

    private static List<DiffEntry> buildDiffs(Map<String, Long> baseMap, Map<String, Long> optMap) {
        Set<String> allKeys = new HashSet<>(baseMap.keySet());
        allKeys.addAll(optMap.keySet());

        List<DiffEntry> diffs = new ArrayList<>();
        for (String key : allKeys) {
            long base = baseMap.getOrDefault(key, 0L);
            long opt = optMap.getOrDefault(key, 0L);
            diffs.add(new DiffEntry(key, base, opt));
        }
        return diffs;
    }

    private static void printDiffTable(String title, List<DiffEntry> diffs, int limit) {
        System.out.println("\n=== " + title + " ===");
        printHeader();
        diffs.stream()
            .sorted((a, b) -> Long.compare(Math.abs(b.delta), Math.abs(a.delta)))
            .limit(limit)
            .forEach(DiffEntry::printRow);
    }

    private static void printHeader() {
        System.out.printf("%-12s | %-12s | %-12s | %-9s | %s%n", "BASELINE", "OPTIMIZED", "DELTA", "CHANGE", "ALLOCATED TYPE / STACK");
        System.out.println("-".repeat(115));
    }

    private static long getEventWeight(RecordedEvent event) {
        if (event.hasField("weight")) return event.getLong("weight");
        if (event.hasField("allocationSize")) return event.getLong("allocationSize");
        if (event.hasField("tlabSize")) return event.getLong("tlabSize");
        return 0L;
    }

    private static double calculatePct(long base, long opt) {
        if (base == 0) return opt > 0 ? 100.0 : 0.0;
        return ((double) (opt - base) / base) * 100.0;
    }

    private static String formatBytes(long bytes) {
        long absBytes = Math.abs(bytes);
        String prefix = bytes < 0 ? "-" : "";
        if (absBytes < 1024) return bytes + " B";
        int exp = (int) (Math.log(absBytes) / Math.log(1024));
        return String.format("%s%.2f %cB", prefix, absBytes / Math.pow(1024, exp), "KMGTPE".charAt(exp - 1));
    }

    private static class JfrData {
        final Map<String, Long> chainAllocations;
        final Map<String, Long> classAllocations;
        final long totalBytes;

        JfrData(Map<String, Long> chain, Map<String, Long> clazz, long totalBytes) {
            this.chainAllocations = chain;
            this.classAllocations = clazz;
            this.totalBytes = totalBytes;
        }
    }

    private static class DiffEntry {
        final String key;
        final long baseline;
        final long optimized;
        final long delta;

        DiffEntry(String key, long baseline, long optimized) {
            this.key = key;
            this.baseline = baseline;
            this.optimized = optimized;
            this.delta = optimized - baseline;
        }

        void printRow() {
            String pctStr;
            if (baseline == 0) {
                pctStr = "NEW";
            } else if (optimized == 0) {
                pctStr = "REMOVED";
            } else {
                pctStr = String.format("%+.1f%%", calculatePct(baseline, optimized));
            }

            System.out.printf("%-12s | %-12s | %-12s | %-9s | %s%n",
                formatBytes(baseline),
                formatBytes(optimized),
                formatBytes(delta),
                pctStr,
                key
            );
        }
    }
}
```