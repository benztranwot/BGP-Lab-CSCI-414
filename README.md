# BGP Lab

We built a network with four autonomous systems using Docker Compose and FRRouting. Left and right advertise destination networks, while top and bottom provide two possible routes between them.

We used this lab to check BGP sessions, change the preferred route with LOCAL_PREF, and observe what happened when we disabled and restored a BGP peer. We also captured the BGP messages in Wireshark.

## Contributors

- Minh Quan Tran
- Evan Brisbin

## Software

We used:

- Docker Desktop running Linux containers
- Docker Compose
- FRRouting 10.7.1, provided by the container image
- Wireshark
- Netshoot for packet capture, downloaded by Docker when needed

FRRouting runs inside the containers. It does not need a separate installation on Windows.

## Files

- `compose.yaml`: routers, virtual links, addresses, and container settings
- `captures/bgp-captures.pcap`: the packet capture used in our report
- `configs/left.conf`: left router configuration and LOCAL_PREF policy
- `configs/top.conf`: top router configuration
- `configs/bottom.conf`: bottom router configuration
- `configs/right.conf`: right router configuration
- `configs/daemons` and `configs/vtysh.conf`: FRRouting startup settings
- `images/`: screenshots from our lab
- `docs/`: project report for our lab

We submit the report, slides, and narrated demonstration separately.

## Topology

| Router | ASN   | BGP router ID | Advertised network |
| ------ | ----- | ------------- | ------------------ |
| left   | 65100 | 10.255.0.1    | 10.51.0.0/24       |
| top    | 65000 | 10.255.0.2    | None               |
| bottom | 65300 | 10.255.0.3    | None               |
| right  | 65200 | 10.255.0.4    | 10.52.0.0/24       |

The two routes between left and right pass through top or bottom.

| Connection   | Subnet       | Router addresses                    |
| ------------ | ------------ | ----------------------------------- |
| left–top     | 10.44.1.0/29 | left: 10.44.1.2, top: 10.44.1.3     |
| top–right    | 10.44.2.0/29 | top: 10.44.2.2, right: 10.44.2.3    |
| left–bottom  | 10.44.3.0/29 | left: 10.44.3.2, bottom: 10.44.3.3  |
| bottom–right | 10.44.4.0/29 | bottom: 10.44.4.2, right: 10.44.4.3 |

Left uses loopback address `10.51.0.1`, and right uses `10.52.0.1`. We use these addresses for the reachability tests.

## Starting the lab

Open Docker Desktop and wait until its engine is running. Open PowerShell in the folder containing `compose.yaml`.

Run:

```powershell
docker compose up -d
docker compose ps
```

Wait about 30 seconds for the routers to establish their BGP sessions.

We disabled reverse path filtering in the containers because outgoing traffic and replies can take different routes. Run this after creating or recreating the containers:

```powershell
foreach ($router in @("left", "top", "bottom", "right")) {
    docker compose exec -T $router sh -c 'for file in /proc/sys/net/ipv4/conf/*/rp_filter; do echo 0 > "$file"; done'
}
```

The Compose file already enables IP forwarding.

Check the sessions and connectivity:

```powershell
docker compose exec left vtysh -c "show bgp ipv4 unicast summary"
docker compose exec left vtysh -c "show bgp ipv4 unicast"
docker compose exec left ping -I 10.51.0.1 -c 3 10.52.0.1
```

Left should have two established neighbors and two candidate routes to `10.52.0.0/24`. The ping should receive replies.

The saved configuration already gives routes from bottom a LOCAL_PREF of 200. To repeat the experiment from the baseline, follow the next section.

## 1. Checking the baseline

Restore the bottom peer and remove its incoming preference policy from the running configuration:

```powershell
docker compose exec -T left vtysh -c "configure terminal" -c "router bgp 65100" -c "no neighbor 10.44.3.3 shutdown" -c "address-family ipv4 unicast" -c "no neighbor 10.44.3.3 route-map PREFER-BOTTOM in" -c "end"
```

Once the peer is established, refresh its routes:

```powershell
docker compose exec -T left vtysh -c "clear bgp 10.44.3.3 in"
```

Check the results:

```powershell
docker compose exec left vtysh -c "show bgp ipv4 unicast summary"
docker compose exec left vtysh -c "show bgp ipv4 unicast 10.52.0.0/24"
docker compose exec left ping -I 10.51.0.1 -c 3 10.52.0.1
```

Both paths should be available with the default LOCAL_PREF of 100:

- Through top: `65000 65200`
- Through bottom: `65300 65200`

In our recorded baseline, top won and FRR reported `best (Router ID)`. A new run can select differently when the earlier attributes are equal. Record the reason FRR actually shows.

## 2. Preferring bottom with LOCAL_PREF

The saved configuration contains this route-map:

```text
route-map PREFER-BOTTOM permit 10
 set local-preference 200
```

Apply it to routes received from bottom:

```powershell
docker compose exec -T left vtysh -c "configure terminal" -c "router bgp 65100" -c "address-family ipv4 unicast" -c "neighbor 10.44.3.3 route-map PREFER-BOTTOM in" -c "end"
docker compose exec -T left vtysh -c "clear bgp 10.44.3.3 in"
docker compose exec left vtysh -c "show bgp ipv4 unicast 10.52.0.0/24"
```

Bottom should become the selected route with `localpref 200` and `best (Local Pref)`. Top keeps the default value of 100.

We changed the preference without changing the AS paths.

## 3. Disabling the preferred peer

Open a second PowerShell terminal in the lab folder and start a continuous ping:

```powershell
docker compose exec left ping -I 10.51.0.1 10.52.0.1
```

Leave it running.

In the first terminal, paste this entire block to disable the bottom peer and measure when our check first observes top as the selected route:

```powershell
$failureTimer = [System.Diagnostics.Stopwatch]::StartNew()

docker compose exec -T left vtysh -c "configure terminal" -c "router bgp 65100" -c "neighbor 10.44.3.3 shutdown"

do {
    $routes = docker compose exec -T left vtysh -c "show bgp ipv4 unicast"
    $usingTop = ($routes -join "`n") -match '(?m)^\s*\*>\s+10\.52\.0\.0/24\s+10\.44\.1\.3\b'
} while (-not $usingTop -and $failureTimer.Elapsed.TotalSeconds -lt 30)

$failureTimer.Stop()
$routes
"Top selected: $usingTop"

if ($usingTop) {
    "Alternate route observed after {0:N2} seconds" -f $failureTimer.Elapsed.TotalSeconds
} else {
    "Top was not observed within 30 seconds. Check the BGP sessions and routes."
}
```

Check the route details:

```powershell
docker compose exec left vtysh -c "show bgp ipv4 unicast 10.52.0.0/24"
```

The selected next hop should be `10.44.1.3`, with AS_PATH `65000 65200`.

This command disables a BGP peer. The Docker network link remains present.

## 4. Restoring the peer

Run:

```powershell
docker compose exec left vtysh -c "configure terminal" -c "router bgp 65100" -c "no neighbor 10.44.3.3 shutdown"
```

Wait for the session to return, then check:

```powershell
docker compose exec left vtysh -c "show bgp ipv4 unicast summary"
docker compose exec left vtysh -c "show bgp ipv4 unicast 10.52.0.0/24"
docker compose exec left ping -I 10.51.0.1 -c 3 10.52.0.1
```

Bottom should win again because its LOCAL_PREF is still 200.

Press `Ctrl+C` in the second terminal to stop the continuous ping and display its summary.

## 5. Inspecting the packet capture

Open `captures/bgp-captures.pcap` in Wireshark.

Use the display filter:

```text
bgp
```

To see all captured traffic using the BGP TCP port, use:

```text
tcp.port == 179
```

These frames are from our submitted capture:

| Frame | Message      | Fields to inspect                                          |
| ----- | ------------ | ---------------------------------------------------------- |
| 13    | NOTIFICATION | Cease 6, Hard Reset 9, data 0602                           |
| 30    | OPEN         | AS 65300, router ID 10.255.0.3, hold time 180              |
| 38    | UPDATE       | AS_PATH 65300 65200, NEXT_HOP 10.44.3.3, NLRI 10.52.0.0/24 |
| 44    | KEEPALIVE    | BGP type 4, message length 19 bytes                        |

Frame 38 contains multiple UPDATE messages. Expand the first BGP section and its NLRI field to see the advertised destination.

The frame numbers will differ in a new capture.

## Recording another capture

We used a separate Netshoot container because the router image did not include tcpdump.

In one PowerShell terminal, run:

```powershell
New-Item -ItemType Directory -Force -Path ".\captures" | Out-Null
$leftContainer = (docker compose ps -q left).Trim()
$captureFolder = (Resolve-Path ".\captures").Path

docker run --rm -it --name bgp-capture --network "container:$leftContainer" --cap-add NET_RAW --cap-add NET_ADMIN --mount "type=bind,source=$captureFolder,target=/captures" nicolaka/netshoot tcpdump -i any -nn -s 0 -U -w /captures/bgp-repeat.pcap "tcp port 179"
```

Wait until tcpdump says it is listening.

In another terminal, disable the bottom peer, wait about 10 seconds, and restore it using the commands above. Wait until BGP establishes the session again.

Press `Ctrl+C` in the capture terminal. Open `captures/bgp-repeat.pcap` in Wireshark.

This saves the new capture separately and preserves the original captures file.

## Our recorded results

We observed:

- Two established BGP peers on left.
- Two available routes to right’s destination network.
- Bottom becoming preferred after we set LOCAL_PREF to 200.
- Top becoming selected after the bottom peer shutdown.
- An observed route switch after approximately 0.96 seconds.
- All 85 monitoring pings receiving replies.
- Bottom becoming preferred again after recovery.
- OPEN, UPDATE, KEEPALIVE, and NOTIFICATION messages in the capture.

The timing includes Docker execution and route-checking overhead. It is an approximate upper bound for the switch we observed.

Our ping sent about one packet per second, so zero observed loss does not rule out a shorter interruption. The packet capture came from a separate repetition of the shutdown and reconnection.

## Stopping the lab

Run:

```powershell
docker compose stop
```

To start it again:

```powershell
docker compose up -d
```

The experiment commands change the running configuration. Recreating a router loads its saved configuration file again. Our saved left configuration prefers bottom with LOCAL_PREF 200.

## Troubleshooting

If Docker reports that it cannot connect to `dockerDesktopLinuxEngine`, open Docker Desktop and wait for the engine to finish starting.

If a service is not running, check:

```powershell
docker compose ps
docker compose logs --tail 50 left
```

If BGP has routes but ping fails, repeat the reverse path filtering commands from the setup section.

The Compose file includes the capabilities our FRRouting image needed to start its routing processes.

## Command reference

[FRRouting BGP documentation](https://docs.frrouting.org/en/stable-10.7/bgp.html) describes the routing policy and route-refresh commands used in this lab.
