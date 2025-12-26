This consistency across your infrastructure (`files resolve [...] dns mdns4_minimal`) is excellent for the hosts, but it presents a specific trap for your "Scratch" containers.

### **The Trap: Active Daemons vs. Static Files**

The `resolve` entry in `nsswitch.conf` tells libc to talk to the `systemd-resolved` daemon (via a socket at `/run/systemd/resolve/...`). The `mdns4_minimal` entry tells libc to talk to the Avahi daemon (multicast DNS).

**In a `FROM scratch` container, these daemons do not exist.**
If you copy the host's `/etc/nsswitch.conf` verbatim into the container, your application will try to connect to non-existent sockets, time out, and potentially crash or hang before falling back to standard DNS.

### **The Solution: Synthetic Configuration**

For the container, we must **simplify** the resolution strategy to strictly `files dns`. This bypasses the need for systemd or Avahi sockets while still allowing your Raku apps to resolve `valkey-vip.rse.local` perfectly using the Docker-provided DNS.

Here is the updated **Builder Logic** to handle this. We will generate a "Container-Optimized" `nsswitch.conf` during the harvest phase.

### **Updated `harvest-libs` Subroutine**

Update your `rse-builder.raku` script with this improved logic. It now creates the configuration artifact `nsswitch.conf` alongside the libraries.

```raku
sub harvest-libs() {
    say "Harvesting runtime libraries and config for Docker scratch images...";
    my $dest = "$TARGET-PREFIX/lib64-runtime";
    
    # Ensure clean state
    if $dest.IO.e {
        # Optional: remove old artifacts
    } else {
        mkdir $dest;
    }
    
    # 1. Capture Essential Libraries
    #    We intentionally EXCLUDE libnss_resolve and libnss_mdns 
    #    because they require active daemons (systemd/avahi).
    my @libs = <
        libc.so.6 libm.so.6 libpthread.so.0 librt.so.1 libdl.so.2 
        ld-linux-x86-64.so.2 
        libresolv.so.2 
        libnss_dns.so.2 
        libnss_files.so.2
        libgcc_s.so.1 libstdc++.so.6
    >;

    for @libs -> $lib {
        my $src = "/lib64/$lib";
        if $src.IO.e {
            copy($src, "$dest/$lib");
        } else {
            note "Warning: System library $lib not found on host!";
        }
    }

    # 2. Generate Synthetic 'nsswitch.conf'
    #    This forces the container to use standard files and DNS only,
    #    ignoring the host's complex systemd-resolved setup.
    my $nss-conf = "$dest/nsswitch.conf";
    $nss-conf.IO.spurt: "hosts: files dns\n";
    say "Generated container-safe nsswitch.conf at $nss-conf";

    say "Library harvest complete at $dest";
}

```

### **Updated Dockerfile Strategy**

Now your Dockerfile doesn't just copy libraries; it injects this "Sanitized" DNS configuration.

```dockerfile
FROM scratch

# 1. Copy the Harvested Runtime Libs
#    (Assumes you rsync'd /var/lib/data/raku/lib64-runtime to ./lib64 in build context)
COPY lib64/*.so* /lib64/

# 2. Inject the Synthetic DNS Configuration
#    This is critical. Without it, glibc defaults can be unpredictable.
COPY lib64/nsswitch.conf /etc/nsswitch.conf

# 3. Environment Environment 
ENV PATH="/var/lib/data/raku/bin:/var/lib/data/raku/share/perl6/site/bin:${PATH}"
ENV LD_LIBRARY_PATH="/lib64:/var/lib/data/raku/lib"
ENV RAKUDO_HOME="/var/lib/data/raku"

WORKDIR /app
COPY bin/helloworld.raku /app/helloworld.raku

# 4. Execution
ENTRYPOINT ["/lib64/ld-linux-x86-64.so.2", "/var/lib/data/raku/bin/raku", "/app/helloworld.raku"]

```

### **Why this works for `valkey-vip.rse.local**`

1. **App Lookup:** `IO::Socket::Async` asks `libc` to resolve the name.
2. **Configuration:** `libc` reads `/etc/nsswitch.conf` (the synthetic one) -> sees `hosts: files dns`.
3. **Files:** It checks `/etc/hosts` (Docker mounts this automatically).
4. **DNS:** It loads `libnss_dns.so.2` (which you copied).
5. **Resolution:** `libnss_dns` reads `/etc/resolv.conf` (Docker mounts this automatically, pointing to your `repository` DNS or internal Docker DNS).
6. **Success:** It resolves `172.19.2.254` instantly without timing out on a missing `systemd-resolved` socket.
