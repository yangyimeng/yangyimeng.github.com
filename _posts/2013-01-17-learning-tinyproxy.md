---
layout: post
title: "Learning tinyproxy"
description: ""
category: 
tags: []
sumarry: |
    In this paper, I just want to record my learning of tinyproxy source code.Just for it!
---
{% include JB/setup %}

####Abstract

In this paper, I just want to record my learning of tinyproxy source code.Just for it!

####Catalogue

1. [about va_list ] (#a1)
2. [about regext] (#a2)
3. [about getopt] (#a3)
4. [about config_parse](#a4)
5. [about shared memory](#a5)
6. [about making daemon](#a6)
7. [about socket struct](#a7)
8. [about select ](#a8)


###Code
I thought the best way to show you the content of va_list is just list the example code.Below is an example of va_list.

<a name ="a1"> </a>

####1. about va_list 

<pre class="brush: js;">
#include &lt;stdio.h&gt;
#include &lt;stdarg.h&gt;
#include &lt;string.h&gt;


/*
We have to know that va_list never provide a method to known how many parameters.So if we want to use va_list, we have to provide a way to known the number of the parameters.So for the function below, we provide the number to known the number.
*/
void log_message(int number,  ...) {
    va_list args;
    int i;
    va_start(args, number);
    const char * ptr;
    for (i = 1; i &lt;= number; i++) {
        ptr = va_arg(args, const char *);
        printf("%s\n", ptr);
    }
    va_end(args);
}


/*
May be you would want to try some way like below the get the number of the parameters, but it failed.So don't think in this way again.
*/
void log_message_v2(int number,  ...) {
    va_list args;
    int i;
    va_start(args, number);
    const char * ptr;
    while ((ptr = va_arg(args, const char *)) != NULL) {
        printf("%s\n", ptr);
    }
    va_end(args);
}


int main() {

    
    log_message(2, "hello world", "I miss you");
    log_message_v2(2, "hello world", "I love you");
    return 0;
}
</pre>



<a name ="a2"> </a>
####2. about regext



<pre class="brush: js;">
#include &lt;stdio.h&gt;
#include &lt;sys/types.h&gt;
#include &lt;regex.h&gt;
#include &lt;string.h&gt;



static char* substr(const char*str, unsigned start, unsigned end)
{
     unsigned n = end - start;
     static char stbuf[256];
     strncpy(stbuf, str + start, n);
     stbuf[n] = 0;
     return stbuf;
}

int main(int argc, char** argv)
{
    char * pattern;
    int x, z, lno = 0, cflags = 0;
    char ebuf[128], lbuf[256];
    regex_t reg;
    regmatch_t pm[10]; 
    const size_t nmatch = 10;

    pattern = argv[1];
    z = regcomp(&amp;reg, pattern, cflags);
    if (z != 0)
    { 
        regerror(z, &amp;reg, ebuf, sizeof(ebuf));
        fprintf(stderr, "%s: pattern '%s' \n",ebuf, pattern);
        return 1;  
    }

     while(fgets(lbuf, sizeof(lbuf), stdin))
    {
        ++lno;
        if ((z = strlen(lbuf)) &gt; 0 &amp;&amp; lbuf[z-1]== '\n') 
            lbuf[z - 1] = 0; 
 
        z = regexec(&amp;reg, lbuf, nmatch, pm, 0);
        if (z == REG_NOMATCH) 
            continue;
        else if (z != 0) {
            regerror(z, &amp;reg, ebuf, sizeof(ebuf));
            fprintf(stderr, "%s: regcom('%s')\n", ebuf, lbuf);
            return 2;
        }

        for (x = 0; x &lt; nmatch &amp;&amp; pm[x].rm_so != -1; ++ x) {
            if (!x) printf("%04d: %s\n", lno, lbuf);
            printf(" $%d='%s'\n", x, substr(lbuf, pm[x].rm_so,pm[x].rm_eo));
        }

    }

    regfree(&amp;reg); 
    return 0;
}
</pre>




<a name ="a3"> </a>
####3. about getopt


<pre class="brush: js;">
#include &lt;stdio.h&gt;
#include &lt;getopt.h&gt;
char *l_opt_arg;
char* const short_options = "n:b:l:";
struct option long_options[] = {
     { "name",     1,   NULL,    'n'     },
     { "bf_name",  1,   NULL,    'b'     },
     { "love",     1,   NULL,    'l'     },
     {      0,     0,     0,     0},
};
int main(int argc, char *argv[])
{
     int c;
     while((c = getopt_long (argc, argv, short_options, long_options, NULL)) != -1)
     {
         switch (c)
         {
         case 'n':
             l_opt_arg = optarg;
             printf("My name is %s. \n", l_opt_arg);
             break;
         case 'b':
             l_opt_arg = optarg;
             printf("His name is %s. \n", l_opt_arg);
             break;
         case 'l':
             l_opt_arg = optarg;
             printf("Our love is %s! \n", l_opt_arg);
             break;
         }
     }
     return 0;
}

</pre>


<a name ="a4"> </a>
####4. about config_parse

First, Below is the code of the function config_parse, Every loop the code read one line,and process this line by involk the function check_match.

<pre class="brush: js;">
/*
 * Parse the previously opened configuration stream.
 */
static int config_parse (struct config_s *conf, FILE * f)
{
        char buffer[1024];      /* 1KB lines should be plenty */
        unsigned long lineno = 1;

        while (fgets (buffer, sizeof (buffer), f)) {
                if (check_match (conf, buffer)) {
                        printf ("Syntax error on line %ld\n", lineno);
                        return 1;
                }
                ++lineno;
        }
        return 0;
}
</pre>


Ok, Let's have a look at the function check_match.And here we can see the important key code.Every loop, the code invoke the function regexec to match .If matched , then invoke relevant function .handler.


<pre class="brush: js;">
/*
 * Attempt to match the supplied line with any of the configuration
 * regexes defined above.  If a match is found, call the handler
 * function to process the directive.
 *
 * Returns 0 if a match was found and successfully processed; otherwise,
 * a negative number is returned.
 */
static int check_match (struct config_s *conf, const char *line)
{
        regmatch_t match[RE_MAX_MATCHES];
        unsigned int i;

        assert (ndirectives &gt; 0);

        for (i = 0; i != ndirectives; ++i) {
                assert (directives[i].cre);
                if (!regexec
                    (directives[i].cre, line, RE_MAX_MATCHES, match, 0))
                        return (*directives[i].handler) (conf, line, match);
        }

        return -1;
}
</pre>


So, you may want to known what is directives .I have't shown its code before.So I list its code below.

<pre class="brush: js;">
int
config_compile_regex (void)
{
        unsigned int i, r;

        for (i = 0; i != ndirectives; ++i) {
                assert (directives[i].handler);
                assert (!directives[i].cre);

                directives[i].cre = (regex_t *) safemalloc (sizeof (regex_t));
                if (!directives[i].cre)
                        return -1;

                r = regcomp (directives[i].cre,
                             directives[i].re,
                             REG_EXTENDED | REG_ICASE | REG_NEWLINE);
                if (r)
                        return r;
        }

        atexit (config_free_regex);

        return 0;
}
</pre>


<a name ="a5"> </a>
####5. about shared memory


Here I list the shared memory means used by this project.Nothing to say, just watch the code.

<pre class="brush: js;">
/*
 * Initialize the statistics information to zero.
 */
void init_stats (void)
{
        stats = (struct stat_s *) malloc_shared_memory (sizeof (struct stat_s));
        if (stats == MAP_FAILED)
                return;

        memset (stats, 0, sizeof (struct stat_s));
}
</pre>


And from the code before,we known the most important function is malloc_shared_memory.Just see it below.

<pre class="brush: js;">
/*
 * Allocate a block of memory in the "shared" memory region.
 *
 * FIXME: This uses the most basic (and slowest) means of creating a
 * shared memory location.  It requires the use of a temporary file.  We might
 * want to look into something like MM (Shared Memory Library) for a better
 * solution.
 */
void *malloc_shared_memory (size_t size)
{
        int fd;
        void *ptr;
        char buffer[32];

        static const char *shared_file = "/tmp/tinyproxy.shared.XXXXXX";
        //here, use the temp file for shared memory.
        assert (size &gt; 0);

        strlcpy (buffer, shared_file, sizeof (buffer));

        /* Only allow u+rw bits. This may be required for some versions
         * of glibc so that mkstemp() doesn't make us vulnerable.
         */
        umask (0177);
        //create the temporary file
        if ((fd = mkstemp (buffer)) == -1)
                return MAP_FAILED;
        unlink (buffer);
        //after creating the temporary file, we need to unlink the file to delete it after application is closed.

        //here truncate the file length to size
        if (ftruncate (fd, size) == -1)
                return MAP_FAILED;
        ptr = mmap (NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        //map the temporary file to the virtual memory space.

        return ptr;
}
</pre>





<a name ="a6"> </a>
####6. about making daemon


Nothing to say, just watch the code below.

<pre class="brush: js;">

/*
 * Fork a child process and then kill the parent so make the calling
 * program a daemon process.
 */
void makedaemon (void)
{
    if (fork () != 0)
        exit (0);

    setsid ();
    set_signal_handler (SIGHUP, SIG_IGN);

    if (fork () != 0)
        exit (0);

    if (chdir ("/") != 0) {
        log_message (LOG_WARNING, "Could not change directory to /");
    }

    umask (0177);
#ifdef NDEBUG
        /*
         * When not in debugging mode, close the standard file
         * descriptors.
         */
        close (0);
        close (1);
        close (2);
#endif
}

</pre>


<a name ="a7"> </a>

####7. about socket struct


###First, let's talk about getaddrinfo 

<pre class="brush: js;">
struct addrinfo {
    int              ai_flags;
    int              ai_family;
    int              ai_socktype;
    int              ai_protocol;
    size_t           ai_addrlen;
    struct sockaddr *ai_addr;
    char            *ai_canonname;
    struct addrinfo *ai_next;
};


if (getaddrinfo (config.ipAddr, portstr, &amp;hints, &amp;result) != 0) {
        log_message (LOG_ERR,
                     "Unable to getaddrinfo() because of %s",
                     strerror (errno));
        return -1;
}
/*
getaddrinfo() returns a list of address structures. Try each 
address until we successfully bind(2). If socket(2)(or bind(2)) 
fails , we (close the socket and) try the next address        
*/
for (rp = result; rp != NULL; rp = rp-&gt;ai_next) {
        listenfd = socket (rp-&gt;ai_family, rp-&gt;ai_socktype,
                           rp-&gt;ai_protocol);
        if (listenfd == -1)
                continue;
        //here, call setsockopt to make listenfd SO_REUSEADDR
        //it is important
        setsockopt (listenfd, SOL_SOCKET, SO_REUSEADDR, &amp;on,
                    sizeof (on));

        if (bind (listenfd, rp-&gt;ai_addr, rp-&gt;ai_addrlen) == 0)
                break;  /* success */

        close (listenfd);
}

</pre>


###Then let's talk about getpeername and getnameinfo

<pre class="brush: js;">
struct sockaddr {
    sa_family_t sa_family;
    char        sa_data[14];
}

/*here we get sockaddr from fd */
if (getpeername (fd, (struct sockaddr *) &amp;sa, &amp;salen) != 0)
        return -1;
/*here is just get ip address string*/
if (get_ip_string ((struct sockaddr *) &amp;sa, ipaddr, IP_LENGTH) == NULL)
        return -1;

/* Get the full host name(such as dns name) */
ret = getnameinfo ((struct sockaddr *) &amp;sa, salen,
                    string_addr, HOSTNAME_LENGTH, NULL, 0, 0);

/*
 * Convert the network address into either a dotted-decimal or an IPv6
 * hex string.
 */
char *get_ip_string (struct sockaddr *sa, char *buf, size_t buflen)
{
        assert (sa != NULL);
        assert (buf != NULL);
        assert (buflen != 0);
        buf[0] = '\0';          /* start with an empty string */

        switch (sa-&gt;sa_family) {
        case AF_INET:
                {
                        struct sockaddr_in *sa_in = (struct sockaddr_in *) sa;
                        /*
                            here, we need to known function inet_ntop
                            this is just used to 
                            inet_ntop - convert IPv4 and IPv6 addresses from binary to text form
                        */
                        inet_ntop (AF_INET, &amp;sa_in-&gt;sin_addr, buf, buflen);
                        break;
                }
        case AF_INET6:
                {
                        struct sockaddr_in6 *sa_in6 =
                            (struct sockaddr_in6 *) sa;

                        inet_ntop (AF_INET6, &amp;sa_in6-&gt;sin6_addr, buf, buflen);
                        break;
                }
        default:
                /* no valid family */
                return NULL;
        }

        return buf;
}
</pre>

Then we have another way to get sock from fd,it would be shown below.We need to known function getsockname and getpeername.

<pre class="brush: js;">
/*
 * Takes a socket descriptor and returns the socket s IP address.
 */
int getsock_ip (int fd, char *ipaddr)
{
        struct sockaddr_storage name;
        socklen_t namelen = sizeof (name);
        assert (fd &gt;= 0);
        if (getsockname (fd, (struct sockaddr *) &amp;name, &amp;namelen) != 0) {
                log_message (LOG_ERR, "getsock_ip: getsockname() error: %s",
                             strerror (errno));
                return -1;
        }
        if (get_ip_string ((struct sockaddr *) &amp;name, ipaddr, IP_LENGTH) ==
            NULL)
                return -1;

        return 0;
}
</pre>


<a name ="a8"> </a>

####8. about select

Here, we will learn how the tinyproxy use select for proxy function.

<pre class="brush: js;">

/*
 * Switch the sockets into nonblocking mode and begin relaying the bytes
 * between the two connections. We continue to use the buffering code
 * since we want to be able to buffer a certain amount for slower
 * connections (as this was the reason why I originally modified
 * tinyproxy oh so long ago...)
 *  - rjkaes
 */
static void relay_connection (struct conn_s *connptr)
{
        fd_set rset, wset;
        struct timeval tv;
        time_t last_access;
        int ret;
        double tdiff;
        int maxfd = max (connptr-&gt;client_fd, connptr-&gt;server_fd) + 1;
        ssize_t bytes_received;

        socket_nonblocking (connptr-&gt;client_fd);
        socket_nonblocking (connptr-&gt;server_fd);

        last_access = time (NULL);

        for (;;) {
                FD_ZERO (&amp;rset);
                FD_ZERO (&amp;wset);

                tv.tv_sec =
                    config.idletimeout - difftime (time (NULL), last_access);
                tv.tv_usec = 0;
                /*tinyproxy modify the select set by the buffer size and decide the select whichkind of event to listen*/
                if (buffer_size (connptr-&gt;sbuffer) &gt; 0)
                        FD_SET (connptr-&gt;client_fd, &amp;wset);
                if (buffer_size (connptr-&gt;cbuffer) &gt; 0)
                        FD_SET (connptr-&gt;server_fd, &amp;wset);
                if (buffer_size (connptr-&gt;sbuffer) &lt; MAXBUFFSIZE)
                        FD_SET (connptr-&gt;server_fd, &amp;rset);
                if (buffer_size (connptr-&gt;cbuffer) &lt; MAXBUFFSIZE)
                        FD_SET (connptr-&gt;client_fd, &amp;rset);

                /*select is the key function here, after calling this function,we can known wether the fd is ok */
                ret = select (maxfd, &amp;rset, &amp;wset, NULL, &amp;tv);

                if (ret == 0) {
                        tdiff = difftime (time (NULL), last_access);
                        if (tdiff &gt; config.idletimeout) {
                                log_message (LOG_INFO,
                                             "Idle Timeout (after select) as %g &gt; %u.",
                                             tdiff, config.idletimeout);
                                return;
                        } else {
                                continue;
                        }
                } else if (ret &lt; 0) {
                        log_message (LOG_ERR,
                                     "relay_connection: select() error \"%s\". "
                                     "Closing connection (client_fd:%d, server_fd:%d)",
                                     strerror (errno), connptr-&gt;client_fd,
                                     connptr-&gt;server_fd);
                        return;
                } else {
                        /*
                         * All right, something was actually selected so mark it.
                         */
                        last_access = time (NULL);
                }

                if (FD_ISSET (connptr-&gt;server_fd, &amp;rset)) {
                        bytes_received =
                            read_buffer (connptr-&gt;server_fd, connptr-&gt;sbuffer);
                        if (bytes_received &lt; 0)
                                break;

                        connptr-&gt;content_length.server -= bytes_received;
                        if (connptr-&gt;content_length.server == 0)
                                break;
                }
                if (FD_ISSET (connptr-&gt;client_fd, &amp;rset)
                    &amp;&amp; read_buffer (connptr-&gt;client_fd, connptr-&gt;cbuffer) &lt; 0) {
                        break;
                }
                if (FD_ISSET (connptr-&gt;server_fd, &amp;wset)
                    &amp;&amp; write_buffer (connptr-&gt;server_fd, connptr-&gt;cbuffer) &lt; 0) {
                        break;
                }
                if (FD_ISSET (connptr-&gt;client_fd, &amp;wset)
                    &amp;&amp; write_buffer (connptr-&gt;client_fd, connptr-&gt;sbuffer) &lt; 0) {
                        break;
                }
        }

        /*
         * Here the server has closed the connection... write the
         * remainder to the client and then exit.
         */
        socket_blocking (connptr-&gt;client_fd);
        while (buffer_size (connptr-&gt;sbuffer) &gt; 0) {
                if (write_buffer (connptr-&gt;client_fd, connptr-&gt;sbuffer) &lt; 0)
                        break;
        }
        shutdown (connptr-&gt;client_fd, SHUT_WR);

        /*
         * Try to send any remaining data to the server if we can.
         */
        socket_blocking (connptr-&gt;server_fd);
        while (buffer_size (connptr-&gt;cbuffer) &gt; 0) {
                if (write_buffer (connptr-&gt;server_fd, connptr-&gt;cbuffer) &lt; 0)
                        break;
        }

        return;
}

</pre>






