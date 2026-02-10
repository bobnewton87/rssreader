# RSS Reader Installation for SDF (NetBSD)

## Overview

`rssreader` is a BSD-friendly terminal RSS/Atom reader designed for minimal Unix environments like SDF. It requires only:
- Perl (base system)
- curl (for fetching feeds)
- lynx/w3m/links (for HTML rendering)

No CPAN modules required for basic operation. Optional Term::ReadKey enables full TUI mode.

## Quick Install

See the **Copy/Paste Installation Block** at the bottom of this file.

## Manual Installation

### 1. Create directories

```sh
mkdir -p ~/bin ~/.rssreader
```

### 2. Save the script

Save `rssreader` to `~/bin/rssreader`

### 3. Make executable

```sh
chmod +x ~/bin/rssreader
```

### 4. Add to PATH

Add to your `~/.profile` or `~/.kshrc` or `~/.bashrc`:

```sh
export PATH="$HOME/bin:$PATH"
```

Then reload:

```sh
. ~/.profile
```

### 5. Create default feeds

The script creates a default feeds file on first run, or you can create it manually:

```sh
cat > ~/.rssreader/feeds << 'EOF'
# RSS Reader Feed List
# Format: URL<TAB>tag
https://tldr.tech/api/rss/tech	tech
https://tldr.tech/api/rss/ai	ai
https://tldr.tech/api/rss/devops	devops
https://tldr.tech/api/rss/crypto	crypto
https://tldr.tech/api/rss/fintech	fintech
https://tldr.tech/api/rss/webdev	dev
EOF
```

## Usage

### Basic Commands

```sh
# Interactive TUI mode
rssreader

# Fetch/reload feeds first
rssreader --reload

# Show only top 30 items
rssreader --top 30

# Show only AI items
rssreader --tag ai

# Show only unread items
rssreader --unread

# Print list to stdout (for scripting)
rssreader --print

# Open item #5 in text browser
rssreader --open 5

# Combined: top 20 unread AI items
rssreader --reload --top 20 --tag ai --unread
```

### Interactive Mode Keys

| Key | Action |
|-----|--------|
| j/k or arrows | Navigate up/down |
| Enter | Open item details |
| o | Open full article in browser |
| r | Reload feeds |
| u | Toggle unread-only filter |
| t | Set top N filter |
| g | Set tag filter |
| m | Mark current item read/unread |
| M | Mark all visible as read |
| Space | Page down |
| b | Page up |
| q | Quit |

## Daily Workflow

Add this to your login routine or crontab:

```sh
# Morning routine - fetch and read top 30 items
rssreader --reload && rssreader --top 30

# Quick check during the day
rssreader --unread

# Focus on specific topic
rssreader --tag ai --unread
```

### Cron job (optional)

Fetch feeds every 4 hours:

```sh
crontab -e
# Add:
0 */4 * * * $HOME/bin/rssreader --reload >/dev/null 2>&1
```

## Adding Feeds

Edit `~/.rssreader/feeds`:

```sh
vi ~/.rssreader/feeds
```

Format: `URL<TAB>tag` (one per line)

Example:
```
https://hnrss.org/frontpage	hackernews
https://lobste.rs/rss	lobsters
https://news.ycombinator.com/rss	hackernews
```

## Troubleshooting

### UTF-8/Locale Issues

Set proper locale before running:

```sh
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
rssreader
```

Add to your `.profile` for permanent fix.

### Term::ReadKey not available

The script will fall back to "simple mode" with numbered prompts. This works fine, just less convenient.

To check if Term::ReadKey is available:

```sh
perl -e 'use Term::ReadKey; print "OK\n"'
```

### Change text browser

Edit the script and modify the `$TEXT_BROWSER` detection, or set the priority by reordering the if/elsif chain.

The script auto-detects in order: lynx, w3m, links

### Curl path

If curl is in a different location:

```sh
which curl
# Then edit the $CURL variable in the script
```

### Feed parsing errors

Some feeds have unusual formats. The parser handles:
- RSS 2.0
- Atom
- CDATA sections
- Various date formats

If a feed doesn't work, check it manually:

```sh
curl -s "https://example.com/feed.xml" | head -50
```

## Files

| Path | Purpose |
|------|---------|
| `~/bin/rssreader` | Main script |
| `~/.rssreader/feeds` | Feed list (URL + tag) |
| `~/.rssreader/items.tsv` | Cached items |
| `~/.rssreader/read.tsv` | Read state tracking |

## Uninstall

```sh
rm ~/bin/rssreader
rm -rf ~/.rssreader
```

---

## Copy/Paste Installation Block

Copy everything between the lines below and paste into your SDF shell:

```sh
# ============= RSS READER INSTALLATION =============
# Creates ~/bin/rssreader and ~/.rssreader/ directory

mkdir -p ~/bin ~/.rssreader

cat > ~/bin/rssreader << 'ENDSCRIPT'
#!/usr/bin/env perl
# rssreader - A nano-style BSD-friendly RSS/Atom reader
use strict;
use warnings;
use POSIX qw(strftime mktime);
use Digest::MD5 qw(md5_hex);
use Fcntl qw(:flock);
use File::Path qw(make_path);
use Getopt::Long;
use Time::Local qw(timegm timelocal);

my $HOME = $ENV{HOME} || (getpwuid($<))[7];
my $CONFIG_DIR = "$HOME/.rssreader";
my $FEEDS_FILE = "$CONFIG_DIR/feeds";
my $ITEMS_FILE = "$CONFIG_DIR/items.tsv";
my $READ_FILE = "$CONFIG_DIR/read.tsv";
my $CURL = '/usr/pkg/bin/curl';
my $LYNX = '/usr/pkg/bin/lynx';
my $W3M = '/usr/pkg/bin/w3m';
my $LINKS = '/usr/pkg/bin/links';

my $TEXT_BROWSER;
if (-x $LYNX) { $TEXT_BROWSER = "$LYNX -dump -stdin -width=80 -nolist"; }
elsif (-x $W3M) { $TEXT_BROWSER = "$W3M -dump -T text/html"; }
elsif (-x $LINKS) { $TEXT_BROWSER = "$LINKS -dump"; }
else { $TEXT_BROWSER = "cat"; }

$CURL = '/usr/bin/curl' unless -x $CURL;
$CURL = 'curl' unless -x $CURL;

my %READ_STATE; my @ITEMS; my @FEEDS;
my $HAS_READKEY = 0; my $TERM_ROWS = 24; my $TERM_COLS = 80;

eval { require Term::ReadKey; Term::ReadKey->import(); $HAS_READKEY = 1; };

my %opts = (reload=>0, top=>0, tag=>'', print=>0, open=>0, unread=>0, help=>0);
GetOptions('reload|r'=>\$opts{reload}, 'top|t=i'=>\$opts{top}, 'tag=s'=>\$opts{tag},
    'print|p'=>\$opts{print}, 'open|o=i'=>\$opts{open}, 'unread|u'=>\$opts{unread},
    'help|h'=>\$opts{help}) or die "Error in arguments\n";

if ($opts{help}) { print_help(); exit 0; }

init_config(); load_read_state(); load_feeds(); load_items();

if ($opts{reload}) { fetch_all_feeds(); load_items(); }
if ($opts{open} > 0) { open_item($opts{open}); exit 0; }
if ($opts{print}) { print_items(); exit 0; }
run_interactive();

sub init_config {
    make_path($CONFIG_DIR) unless -d $CONFIG_DIR;
    unless (-f $FEEDS_FILE) {
        open my $fh, '>', $FEEDS_FILE or die "Cannot create $FEEDS_FILE: $!\n";
        print $fh "https://tldr.tech/api/rss/tech\ttech\n";
        print $fh "https://tldr.tech/api/rss/ai\tai\n";
        print $fh "https://tldr.tech/api/rss/devops\tdevops\n";
        print $fh "https://tldr.tech/api/rss/crypto\tcrypto\n";
        print $fh "https://tldr.tech/api/rss/fintech\tfintech\n";
        print $fh "https://tldr.tech/api/rss/webdev\tdev\n";
        close $fh;
    }
    for my $f ($ITEMS_FILE, $READ_FILE) {
        unless (-f $f) { open my $fh, '>', $f; close $fh; }
    }
}

sub load_feeds {
    @FEEDS = ();
    open my $fh, '<', $FEEDS_FILE or return;
    while (<$fh>) { chomp; next if /^\s*#/ || /^\s*$/;
        my ($url, $tag) = split /\t/, $_, 2; $tag //= 'general';
        push @FEEDS, { url => $url, tag => $tag }; }
    close $fh;
}

sub load_read_state {
    %READ_STATE = ();
    open my $fh, '<', $READ_FILE or return;
    while (<$fh>) { chomp; my ($id, $time) = split /\t/, $_, 2;
        $READ_STATE{$id} = $time if defined $id && defined $time; }
    close $fh;
}

sub save_read_state {
    open my $fh, '>', $READ_FILE or return; flock($fh, LOCK_EX);
    print $fh "$_\t$READ_STATE{$_}\n" for keys %READ_STATE; close $fh;
}

sub load_items {
    @ITEMS = ();
    open my $fh, '<', $ITEMS_FILE or return;
    while (<$fh>) { chomp;
        my ($id, $date, $url, $title, $tag, $desc) = split /\t/, $_, 6;
        next unless defined $id && defined $url;
        push @ITEMS, { id=>$id, date=>$date//'', url=>$url, title=>$title//'(no title)',
            tag=>$tag//'general', desc=>$desc//'', read=>exists $READ_STATE{$id} }; }
    close $fh;
    @ITEMS = sort { ($b->{date}//'') cmp ($a->{date}//'') } @ITEMS;
}

sub save_items {
    open my $fh, '>', $ITEMS_FILE or return; flock($fh, LOCK_EX);
    for my $item (@ITEMS) {
        my $desc = $item->{desc}//''; $desc =~ s/[\t\n\r]/ /g;
        print $fh join("\t", $item->{id}, $item->{date}, $item->{url},
            $item->{title}, $item->{tag}, $desc) . "\n"; }
    close $fh;
}

sub fetch_all_feeds {
    print "Fetching feeds...\n";
    my %seen_ids; my @new_items;
    load_items(); $seen_ids{$_->{id}} = 1 for @ITEMS;
    my $count = 0;
    for my $feed (@FEEDS) {
        $count++;
        print "  [$count/".scalar(@FEEDS)."] $feed->{tag}: $feed->{url}\n";
        my $content = fetch_url($feed->{url});
        if ($content) {
            my @items = parse_feed($content, $feed->{tag});
            for my $item (@items) {
                unless ($seen_ids{$item->{id}}) {
                    push @new_items, $item; $seen_ids{$item->{id}} = 1; } } }
        select(undef, undef, undef, 0.3) if $count < scalar(@FEEDS); }
    if (@new_items) {
        print "Found ".scalar(@new_items)." new items.\n";
        push @ITEMS, @new_items;
        @ITEMS = sort { ($b->{date}//'') cmp ($a->{date}//'') } @ITEMS;
        save_items();
    } else { print "No new items.\n"; }
}

sub fetch_url {
    my ($url) = @_;
    my @cmd = ($CURL, '--silent', '--fail', '--location', '--max-time', '20',
        '--connect-timeout', '10', '--user-agent', 'RSSReader/1.0', $url);
    my $content = '';
    my $pid = open my $fh, '-|';
    return undef unless defined $pid;
    if ($pid == 0) { exec(@cmd) or exit 1; }
    { local $/; $content = <$fh>; } close $fh;
    return $? == 0 ? $content : undef;
}

sub parse_feed {
    my ($content, $tag) = @_;
    if ($content =~ /<feed/i) { return parse_atom($content, $tag); }
    elsif ($content =~ /<rss|<channel/i) { return parse_rss($content, $tag); }
    return ();
}

sub parse_rss {
    my ($content, $tag) = @_; my @items;
    while ($content =~ /<item[^>]*>(.*?)<\/item>/gis) {
        my $x = $1;
        my $title = extract_tag($x,'title') // '(no title)';
        my $link = extract_tag($x,'link') // '';
        my $guid = extract_tag($x,'guid') // '';
        my $pubdate = extract_tag($x,'pubDate') // '';
        my $desc = extract_tag($x,'description') // '';
        $title = decode_entities($title); $title =~ s/^\s+|\s+$//g; $title =~ s/\s+/ /g;
        $link =~ s/^\s+|\s+$//g;
        $desc = decode_entities($desc); $desc =~ s/<[^>]+>//g;
        $desc =~ s/^\s+|\s+$//g; $desc = substr($desc,0,500) if length($desc)>500;
        my $id = $guid || md5_hex($link.$title);
        my $iso = normalize_date($pubdate);
        push @items, { id=>$id, date=>$iso, url=>$link, title=>$title,
            tag=>$tag, desc=>$desc, read=>0 } if $link;
    }
    return @items;
}

sub parse_atom {
    my ($content, $tag) = @_; my @items;
    while ($content =~ /<entry[^>]*>(.*?)<\/entry>/gis) {
        my $x = $1;
        my $title = extract_tag($x,'title') // '(no title)';
        my $id = extract_tag($x,'id') // '';
        my $updated = extract_tag($x,'updated') // '';
        my $published = extract_tag($x,'published') // $updated;
        my $summary = extract_tag($x,'summary') // extract_tag($x,'content') // '';
        my $link = '';
        if ($x =~ /<link[^>]*href=["']([^"']+)["']/) { $link = $1; }
        elsif ($x =~ /<link[^>]*>([^<]+)<\/link>/) { $link = $1; }
        $title = decode_entities($title); $title =~ s/^\s+|\s+$//g; $title =~ s/\s+/ /g;
        $link =~ s/^\s+|\s+$//g;
        $summary = decode_entities($summary); $summary =~ s/<[^>]+>//g;
        $summary =~ s/^\s+|\s+$//g; $summary = substr($summary,0,500) if length($summary)>500;
        $id = md5_hex($link.$title) unless $id;
        my $iso = normalize_date($published || $updated);
        push @items, { id=>$id, date=>$iso, url=>$link, title=>$title,
            tag=>$tag, desc=>$summary, read=>0 } if $link;
    }
    return @items;
}

sub extract_tag {
    my ($xml, $tag) = @_;
    if ($xml =~ /<$tag[^>]*>(?:<!\[CDATA\[(.*?)\]\]>|([^<]*))<\/$tag>/is) { return $1 // $2; }
    if ($xml =~ /<$tag[^>]*>(.*?)<\/$tag>/is) { return $1; }
    return undef;
}

sub decode_entities {
    my ($s) = @_; return '' unless defined $s;
    $s =~ s/&lt;/</g; $s =~ s/&gt;/>/g; $s =~ s/&amp;/&/g;
    $s =~ s/&quot;/"/g; $s =~ s/&apos;/'/g;
    $s =~ s/&#(\d+);/chr($1)/ge; $s =~ s/&#x([0-9a-fA-F]+);/chr(hex($1))/ge;
    return $s;
}

sub normalize_date {
    my ($d) = @_; return '' unless $d;
    if ($d =~ /^(\d{4}-\d{2}-\d{2})T(\d{2}:\d{2}:\d{2})/) { return "$1T$2Z"; }
    my %m = (Jan=>'01',Feb=>'02',Mar=>'03',Apr=>'04',May=>'05',Jun=>'06',
             Jul=>'07',Aug=>'08',Sep=>'09',Oct=>'10',Nov=>'11',Dec=>'12');
    if ($d =~ /(\d{1,2})\s+(\w{3})\s+(\d{4})\s+(\d{2}):(\d{2}):(\d{2})/) {
        my ($day,$mon,$yr,$h,$m,$s) = ($1,$2,$3,$4,$5,$6);
        $mon = $m{$mon} // '01';
        return sprintf("%04d-%02d-%02dT%02d:%02d:%02dZ",$yr,$mon,$day,$h,$m,$s);
    }
    if ($d =~ /(\d{4})-(\d{2})-(\d{2})/) { return "$1-$2-$3T00:00:00Z"; }
    return strftime("%Y-%m-%dT%H:%M:%SZ", gmtime());
}

sub get_filtered_items {
    my @f = @ITEMS;
    @f = grep { lc($_->{tag}) eq lc($opts{tag}) } @f if $opts{tag};
    @f = grep { !$_->{read} } @f if $opts{unread};
    @f = @f[0..$opts{top}-1] if $opts{top} > 0 && @f > $opts{top};
    return @f;
}

sub print_items {
    my @f = get_filtered_items(); my $n = 0;
    for my $item (@f) {
        $n++; my $rm = $item->{read} ? ' ' : '*';
        my $dt = substr($item->{date},0,10);
        printf "%3d%s [%s] [%-8s] %s\n", $n, $rm, $dt, $item->{tag}, $item->{title};
    }
    print "\nTotal: $n items\n" if $n;
}

sub open_item {
    my ($num) = @_; my @f = get_filtered_items();
    die "Invalid: $num (1-".scalar(@f).")\n" if $num < 1 || $num > @f;
    my $item = $f[$num-1];
    print "Opening: $item->{title}\nURL: $item->{url}\n\n";
    my $content = fetch_url($item->{url});
    if ($content) {
        open my $b, '|-', "$TEXT_BROWSER | less -R" or die "Cannot pipe: $!";
        print $b $content; close $b;
    } else { print "Failed to fetch.\n"; }
    mark_read($item->{id});
}

sub mark_read {
    my ($id) = @_; $READ_STATE{$id} = time();
    $_->{read} = 1 for grep { $_->{id} eq $id } @ITEMS;
    save_read_state();
}

sub mark_unread {
    my ($id) = @_; delete $READ_STATE{$id};
    $_->{read} = 0 for grep { $_->{id} eq $id } @ITEMS;
    save_read_state();
}

sub run_interactive { $HAS_READKEY ? run_tui() : run_simple_ui(); }

sub run_simple_ui {
    print "\n=== RSS Reader (Simple Mode) ===\n";
    my $ps = 20; my $off = 0;
    while (1) {
        my @f = get_filtered_items(); my $tot = scalar(@f);
        print "\n" . "=" x 70 . "\n";
        my $ti = $opts{tag} ? " [tag:$opts{tag}]" : "";
        my $ui = $opts{unread} ? " [unread]" : "";
        print "Items ".($off+1)."-".min($off+$ps,$tot)." of $tot$ti$ui\n";
        print "=" x 70 . "\n\n";
        for my $i ($off..min($off+$ps-1,$tot-1)) {
            my $it = $f[$i]; my $n = $i+1;
            my $rm = $it->{read} ? ' ' : '*';
            my $dt = substr($it->{date},0,10);
            printf "%3d%s [%s] [%-8s] %s\n", $n,$rm,$dt,$it->{tag},substr($it->{title},0,50);
        }
        print "\n[num] open | n/p page | r reload | u unread | t top | g tag | q quit\n> ";
        my $in = <STDIN>; last unless defined $in;
        chomp $in; $in =~ s/^\s+|\s+$//g;
        if ($in eq 'q') { last; }
        elsif ($in eq 'n') { $off += $ps if $off+$ps < $tot; }
        elsif ($in eq 'p') { $off -= $ps; $off = 0 if $off < 0; }
        elsif ($in eq 'r') { fetch_all_feeds(); load_items(); $off = 0; }
        elsif ($in eq 'u') { $opts{unread} = !$opts{unread}; $off = 0; }
        elsif ($in =~ /^t\s*(\d*)$/) {
            if ($1) { $opts{top} = $1; }
            else { print "Top N: "; my $n = <STDIN>; chomp $n;
                   $opts{top} = int($n) if $n =~ /^\d+$/; }
            $off = 0;
        }
        elsif ($in =~ /^g\s*(\w*)$/) {
            if ($1) { $opts{tag} = $1; }
            else { my %tags; $tags{$_->{tag}}++ for @ITEMS;
                   print "Tags: ".join(", ",sort keys %tags)."\nTag: ";
                   my $t = <STDIN>; chomp $t; $opts{tag} = $t; }
            $off = 0;
        }
        elsif ($in =~ /^(\d+)$/) {
            my $n = $1;
            if ($n >= 1 && $n <= $tot) { open_item_int($f[$n-1]); }
            else { print "Invalid: 1-$tot\n"; }
        }
    }
    print "Goodbye!\n";
}

sub open_item_int {
    my ($it) = @_;
    print "\n" . "=" x 70 . "\n";
    print "Title: $it->{title}\nDate:  $it->{date}\nTag:   $it->{tag}\nURL:   $it->{url}\n";
    print "=" x 70 . "\n";
    print "\n$it->{desc}\n" if $it->{desc};
    print "\n[o] Open | [m] Toggle read | [Enter] Back\n> ";
    my $in = <STDIN>; chomp $in if defined $in;
    if ($in && $in eq 'o') {
        print "\nFetching...\n";
        my $c = fetch_url($it->{url});
        if ($c) { open my $b,'|-',"$TEXT_BROWSER | less -R"; print $b $c; close $b; }
        else { print "Failed.\n"; <STDIN>; }
        mark_read($it->{id});
    } elsif ($in && $in eq 'm') {
        if ($it->{read}) { mark_unread($it->{id}); print "Unread.\n"; }
        else { mark_read($it->{id}); print "Read.\n"; }
    }
}

sub run_tui {
    eval { ($TERM_COLS,$TERM_ROWS) = Term::ReadKey::GetTerminalSize(); };
    $TERM_ROWS ||= 24; $TERM_COLS ||= 80;
    my ($sel,$off,$msg,$run) = (0,0,"",1);
    Term::ReadKey::ReadMode('cbreak'); $| = 1;
    eval {
        while ($run) {
            my @f = get_filtered_items(); my $tot = scalar(@f);
            my $lh = $TERM_ROWS - 6;
            $off = $sel if $sel < $off;
            $off = $sel - $lh + 1 if $sel >= $off + $lh;
            print "\033[2J\033[H";
            my $ti = $opts{tag} ? " [$opts{tag}]" : "";
            my $ui = $opts{unread} ? " [unread]" : "";
            print "\033[7m RSS Reader$ti$ui" . " " x ($TERM_COLS - 12 - length($ti) - length($ui)) . "\033[0m\n";
            if ($tot == 0) { print "\n  No items. Press 'r' to reload.\n"; }
            else {
                for my $i ($off..min($off+$lh-1,$tot-1)) {
                    my $it = $f[$i]; my $is = ($i == $sel);
                    my $rm = $it->{read} ? ' ' : '*';
                    my $dt = substr($it->{date},0,10);
                    my $tg = sprintf("%-8s", substr($it->{tag},0,8));
                    my $tt = substr($it->{title},0,$TERM_COLS-30);
                    my $ln = sprintf("%s%3d %s [%s] %s", $rm,$i+1,$dt,$tg,$tt);
                    if ($is) { print "\033[7m$ln"." " x ($TERM_COLS-length($ln)-1)."\033[0m\n"; }
                    else { print "$ln\n"; }
                }
            }
            for (1..($lh - min($lh,$tot-$off))) { print "\n"; }
            print "\n";
            if ($msg) { print "\033[33m$msg\033[0m\n"; $msg = ""; }
            else { print "Item ".($sel+1)."/$tot | ".($tot - grep{$_->{read}}@f)." unread\n"; }
            print "\033[7m j/k:nav Enter:open r:reload u:unread t:top g:tag m:mark q:quit \033[0m";
            my $k = Term::ReadKey::ReadKey(0); next unless defined $k;
            if ($k eq 'q') { $run = 0; }
            elsif ($k eq 'j' || ord($k)==66) { $sel++ if $sel < $tot-1; }
            elsif ($k eq 'k' || ord($k)==65) { $sel-- if $sel > 0; }
            elsif ($k eq 'u') { $opts{unread}=!$opts{unread}; $sel=0; $off=0;
                $msg = $opts{unread} ? "Unread only" : "All items"; }
            elsif ($k eq 't') {
                print "\033[2J\033[HTop N (0=all): ";
                Term::ReadKey::ReadMode('normal');
                my $n = <STDIN>; chomp $n;
                Term::ReadKey::ReadMode('cbreak');
                $opts{top} = int($n) if $n =~ /^\d+$/;
                $sel=0; $off=0;
            }
            elsif ($k eq 'g') {
                print "\033[2J\033[H";
                my %tg; $tg{$_->{tag}}++ for @ITEMS;
                print "Tags: ".join(", ",sort keys %tg)."\nTag (empty=all): ";
                Term::ReadKey::ReadMode('normal');
                my $t = <STDIN>; chomp $t;
                Term::ReadKey::ReadMode('cbreak');
                $opts{tag} = $t; $sel=0; $off=0;
            }
            elsif ($k eq 'r') {
                $msg = "Reloading..."; print "\033[2J\033[H$msg\n";
                fetch_all_feeds(); load_items();
                $msg = "Reloaded"; $sel=0; $off=0;
            }
            elsif ($k eq 'm' && $tot > 0) {
                my $it = $f[$sel];
                if ($it->{read}) { mark_unread($it->{id}); $msg = "Unread"; }
                else { mark_read($it->{id}); $msg = "Read"; }
            }
            elsif ($k eq 'M') {
                mark_read($_->{id}) for grep { !$_->{read} } @f;
                $msg = "All marked read";
            }
            elsif ($k eq "\n" || $k eq "\r") {
                if ($tot > 0) { show_item_tui($f[$sel]); mark_read($f[$sel]->{id}); }
            }
            elsif ($k eq ' ') { $sel = min($sel+$lh, $tot-1); }
            elsif ($k eq 'b') { $sel = max($sel-$lh, 0); }
            elsif (ord($k)==27) {
                my $seq = ''; while (my $c = Term::ReadKey::ReadKey(-1)) { $seq .= $c; }
                if ($seq eq '[A') { $sel-- if $sel > 0; }
                elsif ($seq eq '[B') { $sel++ if $sel < $tot-1; }
            }
        }
    };
    Term::ReadKey::ReadMode('normal');
    print "\033[2J\033[HGoodbye!\n";
}

sub show_item_tui {
    my ($it) = @_;
    print "\033[2J\033[H\033[7m ".substr($it->{title},0,$TERM_COLS-2)." \033[0m\n";
    print "Date: $it->{date} | Tag: $it->{tag}\nURL: $it->{url}\n"."-"x$TERM_COLS."\n";
    print "\n$it->{desc}\n" if $it->{desc};
    print "\n\n[o] Open article | [Enter] Back\n";
    my $k = Term::ReadKey::ReadKey(0);
    if (defined $k && $k eq 'o') {
        print "\nFetching...\n";
        Term::ReadKey::ReadMode('normal');
        my $c = fetch_url($it->{url});
        if ($c) { open my $fh,'|-',"$TEXT_BROWSER | less -R"; print $fh $c; close $fh; }
        else { print "Failed.\n"; <STDIN>; }
        Term::ReadKey::ReadMode('cbreak');
    }
}

sub min { $_[0] < $_[1] ? $_[0] : $_[1] }
sub max { $_[0] > $_[1] ? $_[0] : $_[1] }

sub print_help {
    print <<'EOF';
rssreader - BSD-friendly terminal RSS reader

USAGE: rssreader [OPTIONS]

OPTIONS:
  -r, --reload     Fetch feeds
  -t, --top N      Show top N items
      --tag TAG    Filter by tag
  -u, --unread     Show unread only
  -p, --print      Print list to stdout
  -o, --open N     Open item N
  -h, --help       Show help

KEYS (interactive):
  j/k, arrows  Navigate
  Enter        Open item
  o            Open in browser
  r            Reload feeds
  u            Toggle unread
  t            Set top N
  g            Set tag filter
  m            Toggle read/unread
  M            Mark all read
  q            Quit

FILES:
  ~/.rssreader/feeds      Feed URLs (URL<TAB>tag)
  ~/.rssreader/items.tsv  Cached items
  ~/.rssreader/read.tsv   Read state
EOF
}
ENDSCRIPT

chmod +x ~/bin/rssreader

# Create default feeds
cat > ~/.rssreader/feeds << 'EOF'
# TLDR Newsletter RSS feeds
https://tldr.tech/api/rss/tech	tech
https://tldr.tech/api/rss/ai	ai
https://tldr.tech/api/rss/devops	devops
https://tldr.tech/api/rss/crypto	crypto
https://tldr.tech/api/rss/fintech	fintech
https://tldr.tech/api/rss/webdev	dev
# Add more feeds below in format: URL<TAB>tag
EOF

# Touch state files
touch ~/.rssreader/items.tsv ~/.rssreader/read.tsv

# Add to PATH if not already
grep -q 'HOME/bin' ~/.profile 2>/dev/null || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.profile

echo ""
echo "=== Installation Complete ==="
echo ""
echo "To start using rssreader:"
echo "  1. Reload your profile:  . ~/.profile"
echo "  2. Fetch feeds:          rssreader --reload"
echo "  3. Read news:            rssreader"
echo ""
echo "For UTF-8 support, add to ~/.profile:"
echo '  export LANG=en_US.UTF-8'
echo '  export LC_ALL=en_US.UTF-8'
echo ""
# ============= END INSTALLATION =============
```

---

## Note on "The New Paper"

As of this writing, "The New Paper" does not appear to have a publicly documented RSS feed. To add it later if one becomes available:

1. Find their RSS/Atom feed URL
2. Edit `~/.rssreader/feeds`
3. Add: `https://thenewpaper.co/rss.xml	news` (adjust URL as needed)

Check their website footer or `/feed`, `/rss`, `/atom.xml` paths for discovery.
