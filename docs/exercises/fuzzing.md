# Esercizi su fuzzing {#fuzzing-exercises}

Nei seguenti esercizi, viene richiesto di applicare il fuzzing per riprodurre la [nota vulnerabilità `Heartbleed` (CVE-2014-0160)](https://www.heartbleed.com), utilizzando il fuzzer AFL e il sanitizer ASan.

La vulnerabilità Heartbleed è un buffer over-read nel componente Heartbeat Extension di OpenSSL (versione 1.0.1f). OpenSSL è una libreria che implementa i protocolli TLS (Transport Layer Security) ed SSL (Secure Socket Layer).

![XKCD](https://imgs.xkcd.com/comics/heartbleed_explanation.png)

Il codice dell'esercizio è disponibile nella macchina virtuale nella cartella `swsec-labs/fuzzing/heartbleed`, e nel repository online su <https://github.com/swsec-book/swsec-labs/>. L'autore dell'esercizio è Michael Macnair (<https://mykter.com/>).

## Rilevare la vulnerabilità Heartbleed

Il codice contiene una sotto-cartella `openssl` con il codice sorgente vulnerabile.
Il primo passo dell'esercizio richiede di compilare il progetto OpenSSL, utilizzando il compilatore di AFL, e attivando ASan.

```
$ cd openssl
$ <inserire le variabili di ambiente CC, CXX> ./config -d -g -no-shared
$ <inserire le variabili di ambiente per ASAN> make build_libs
```

Si suggerisce di utilizzare i compilatori `afl-clang-fast` (variabile `CC`) e `afl-clang-fast++` (variabile `CXX`).

Il secondo passo dell'esercizio richiede di completare il codice di un programma nel file `handshake.cc` (*test harness*) che richiami la libreria OpenSSL. Il programma dovrà [leggere in input 100 byte dal canale standard input, e copiarli in un array](https://stackoverflow.com/questions/15883568/reading-from-stdin) che sarà passato in input alla libreria OpenSSL.

```
int main() {
  static SSL_CTX *sctx = Init();
  SSL *server = SSL_new(sctx);
  BIO *sinbio = BIO_new(BIO_s_mem());
  BIO *soutbio = BIO_new(BIO_s_mem());
  SSL_set_bio(server, sinbio, soutbio);
  SSL_set_accept_state(server);

  /* TODO: Completare il programma in questo punto,
           leggendo 100 byte dallo STDIN usando la funzione read(),
           e copiando i byte in un array data[100].
           Modificare la chiamata a BIO_write() indicando la dimensione dell'array.
   */

  BIO_write(sinbio, data, size);

  SSL_do_handshake(server);
  SSL_free(server);
  return 0;
}
```


Compilare il test harness, utilizzando il compilatore fornito da AFL, e abilitando ASan.

```
$ <inserire variabile per ASan> <compilatore AFL C++> handshake.cc 
            -o handshake
            openssl/libssl.a openssl/libcrypto.a 
            -I openssl/include -ldl
```

Prima di eseguire il programma, si crei un certificato fittizio con il comando `openssl`.

```
$ openssl req -x509 -newkey rsa:512 -keyout server.key -out server.pem -days 365 -nodes -subj /CN=a/
```

Infine, si avvii il fuzzer:

1. Creare una cartella `input`, che conterrà i seed input per il fuzzing
2. Aggiungere come seed input un singolo file con del semplice testo (ad esempio, la parola "ciao")
3. Lanciare il comando `afl-fuzz`, indicando il percorso del programma `handshake` (ossia il test harness)
4. Attendere un breve tempo (circa 5 minuti), verificare che AFL rilevi almeno un crash

La sintassi del comando `afl-fuzz` è:

```
$ afl-fuzz <parametri> -- <programma target>
```

*Nota:* Fare attenzione a NON usare @@, in modo che i fuzz input siano passati via STDIN.


## Diagnosticare la vulnerabilità

L'esercizio chiede di diagnosticare la vulnerabilità con le informazioni fornite da ASan. 
Eseguire nuovamente il programma `handshake` senza usare AFL, e passandogli in ingresso il file con l'input che causa il crash.

Interpretare l'output di ASan:

- che tipo di errore che è stato rilevato?
- qual è il punto nel codice di OpenSSL che causa l'errore?
- qual è il punto nel codice di OpenSSL che ha allocato il buffer?

Successivamente, si esegua ancora una volta il programma `handshake` tramite GDB, per ottenere ulteriori informazioni di debug. Poiché è abilitato ASan, occorre inserire un breakpoint prima di lanciare l'esecuzione, che ferma il processo prima che ASan forzi la terminazione. Si usi GDB per Ispezionare lo stack frame, ed il contenuto della variabile `payload`.

```
gdb> set breakpoint pending on
gdb> break __asan::ReportGenericError
gdb> run < file_input_crash
...
gdb> backtrace
gdb> frame 2
gdb> print/x payload
```

Si confronti il contenuto della variabile `payload` con il contenuto del file di input.
È possibile visualizzare il file di input con il seguente comando.

```
$ hexdump -C file_input_crash
```

Quali sono i byte dell'input che provocano l'errore?


## ASan e Valgrind

Si ricompili il programma `handshake` *senza ASan*, e lo si esegua di nuovo con l'input che ha prima causato il crash.

In questo caso, il crash non si verificherà di nuovo. Come mai?

Successivamente, si esegua questa version del programma *senza ASan* tramite il tool Valgrind.

In questo caso, la vulnerabilità viene rilevata? Si confronti la diagnosi di Valgrind con quella precedente ottenuta da ASan.

## Persistent mode

Si modifichi il test harness per fare in modo di utilizzare la modalità "persistent mode" di AFL.
Si raccomanda di utilizzare il compilatore `afl-clang-fast++`.
Si consulti la documentazione di AFL per maggiori informazioni, al sito <https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md>.

Si confronti il throughput del fuzzing (numero di esecuzioni al secondo) con e senza la modalità persistent mode.

- Di quanto cambia il throughput?
- Quanto tempo occorre per rilevare la vulnerabilità?

## Correggere la vulnerabilità Heartbleed

La vulnerabilità è legato alla seguente struttura dati, definita in `include/openssl/ssl3.h` del sorgente di OpenSSL.

```
typedef struct ssl3_record_st
        {
/*r */  int type;               /* type of record */
/*rw*/  unsigned int length;    /* How many bytes available */
/*r */  unsigned int off;       /* read/write offset into 'buf' */
/*rw*/  unsigned char *data;    /* pointer to the record data */
/*rw*/  unsigned char *input;   /* where the decode bytes are */
/*r */  unsigned char *comp;    /* only used with decompression - malloc()ed */
/*r */  unsigned long epoch;    /* epoch number, needed by DTLS1 */
/*r */  unsigned char seq_num[8]; /* sequence number, needed by DTLS1 */
        } SSL3_RECORD;
```

Il vettore `data` è strutturato come segue.

| type     | payload_length | ...dati...  | ...padding                     |
| -------- | -------------- | ----------- | ------------------------------ |
| (1 byte) | (2 byte)       | (variabile) | (arrotonda la dimensione a 16) |

Nella struttura `SSL3_RECORD`, il valore di `length` deve essere maggiore o uguale a `payload_length + 1+2+16`.

Correggere il seguente codice vulnerabile in OpenSSL per prevenire l'attacco. Se `payload_length` contiene un valore eccessivamente elevato, si forzi l'uscita della funzione con il valore zero (`return 0`).

```
    unsigned char *p = &s->s3->rrec.data[0], *pl;
    unsigned short hbtype;
    unsigned int payload;
    unsigned int padding = 16; /* Use minimum padding */

    /* Read type and payload length first */
    hbtype = *p++;
    n2s(p, payload);

    /* TODO: Aggiungere input validation */

    ...
    /* Enter response type, length and copy payload */
    *bp++ = TLS1_HB_RESPONSE;
    s2n(payload, bp);
    memcpy(bp, pl, payload);
```

