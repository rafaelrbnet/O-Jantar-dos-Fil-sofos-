# Relatório do Experimento  
## O Jantar dos Filósofos – Concorrência, Sincronização e Deadlocks em Python  

**Aluno:** Rafael Baena Neto  
**Prontuário:** GU3066916  
**Disciplina:** POS.01178 (C1AOC) – Arquitetura, Organização e Redes de Computadores  

---

### 1. Introdução  

O “Jantar dos Filósofos” é um problema clássico de sincronização: cinco filósofos, sentados em torno de uma mesa redonda, alternam entre pensar e comer espaguete.  
Para comer, cada filósofo precisa de **dois garfos** – o à esquerda e o à direita – mas cada garfo é compartilhado entre vizinhos.  
Se todos pegarem o garfo à esquerda ao mesmo tempo, ninguém consegue o segundo garfo e o **deadlock** acontece.  

Compreender e evitar deadlocks é crucial em sistemas concorrentes, de bancos de dados a servidores web.

---

### 2. Fase&nbsp;1 – Implementação Ingênua (Deadlock Garantido)  

#### 2.1&nbsp;Código‑fonte  

```python
#!/usr/bin/env python3
"""Fase 1 – implementa o jantar dos filósofos forçando deadlock."""
import threading, time, random

NUM_FILOSOFOS = 5
MIN_PENSAR, MAX_PENSAR = 0.2, 0.6
MIN_COMER,  MAX_COMER  = 0.2, 0.6

garfos = [threading.Lock() for _ in range(NUM_FILOSOFOS)]
todos_primeiro = threading.Semaphore(0)
pode_segundo   = threading.Semaphore(0)

def pensar(fid):
    print(f"Φ{fid} está PENSANDO")
    time.sleep(random.uniform(MIN_PENSAR, MAX_PENSAR))

def comer(fid):
    print(f"Φ{fid} está COMENDO 🍝")
    time.sleep(random.uniform(MIN_COMER, MAX_COMER))

def filosofo(fid):
    g_esq = fid
    g_dir = (fid + 1) % NUM_FILOSOFOS
    while True:
        pensar(fid)

        print(f"Φ{fid} tenta pegar garfo {g_esq} (esq)")
        garfos[g_esq].acquire()
        print(f"Φ{fid} pegou garfo {g_esq}")

        todos_primeiro.release()

        for _ in range(NUM_FILOSOFOS):
            pode_segundo.acquire()
        for _ in range(NUM_FILOSOFOS):
            pode_segundo.release()

        print(f"Φ{fid} tenta pegar garfo {g_dir} (dir)")
        garfos[g_dir].acquire()   # BLOQUEIO aqui!
        print(f"Φ{fid} pegou garfo {g_dir}")

        comer(fid)

        garfos[g_esq].release()
        garfos[g_dir].release()
        print(f"Φ{fid} devolveu garfos {g_esq} e {g_dir}")

def coordenador():
    while True:
        for _ in range(NUM_FILOSOFOS):
            todos_primeiro.acquire()
        for _ in range(NUM_FILOSOFOS):
            pode_segundo.release()

if __name__ == "__main__":
    random.seed(time.time())
    threading.Thread(target=coordenador, daemon=True).start()
    threads = [threading.Thread(target=filosofo, args=(i,), daemon=True)
               for i in range(NUM_FILOSOFOS)]
    for t in threads: t.start()
    for t in threads: t.join()
```

#### 2.2&nbsp;Observações e Análise  

* **Comportamento observado:** após todos pegarem o garfo esquerdo, o programa congela tentando adquirir o direito.  
* **Últimas mensagens:** `Φi pegou garfo k` e `Φi tenta pegar garfo k+1`.  

| Condição de Deadlock | Evidência no código/execução |
|----------------------|------------------------------|
| Exclusão mútua       | Cada `Lock()` protege um único garfo. |
| Posse e espera       | Cada filósofo segura 1 garfo e espera o outro. |
| Não preempção        | O garfo não pode ser tomado à força. |
| Espera circular      | Φ0→garfo1, Φ1→garfo2, … Φ4→garfo0. |

#### 2.3&nbsp;Conclusão da Fase 1  

A simetria de comportamento criou um ciclo de dependências inquebrável – demostrando que satisfazer as quatro condições leva inevitavelmente ao deadlock.

---

### 3. Fase&nbsp;2 – Implementação com Solução (Sem Deadlock)  

#### 3.1&nbsp;Código‑fonte  

```python
#!/usr/bin/env python3
"""Fase 2 – solução: filósofo 0 inverte a ordem dos garfos."""
import threading, time, random

NUM_FILOSOFOS = 5
MIN_PENSAR, MAX_PENSAR = 0.2, 0.6
MIN_COMER,  MAX_COMER  = 0.2, 0.6

garfos = [threading.Lock() for _ in range(NUM_FILOSOFOS)]

def pensar(fid):
    print(f"Φ{fid} está PENSANDO")
    time.sleep(random.uniform(MIN_PENSAR, MAX_PENSAR))

def comer(fid):
    print(f"Φ{fid} está COMENDO 🍝")
    time.sleep(random.uniform(MIN_COMER, MAX_COMER))

def filosofo(fid):
    g_esq = fid
    g_dir = (fid + 1) % NUM_FILOSOFOS
    while True:
        pensar(fid)
        primeiro, segundo = (g_dir, g_esq) if fid == 0 else (g_esq, g_dir)

        garfos[primeiro].acquire()
        garfos[segundo].acquire()

        comer(fid)

        garfos[segundo].release()
        garfos[primeiro].release()

if __name__ == "__main__":
    random.seed(time.time())
    threads = [threading.Thread(target=filosofo, args=(i,), daemon=True)
               for i in range(NUM_FILOSOFOS)]
    for t in threads: t.start()
    for t in threads: t.join()
```

#### 3.2&nbsp;Observações e Análise  

* O programa executa indefinidamente sem travar; filósofos alternam pensar/comer.  
* **Estratégia:** quebra da simetria – filósofo 0 pega o garfo direito primeiro.  
* **Condição eliminada:** *espera circular* deixa de existir ⇒ deadlock impossível.  
* **Inanição:** improvável sob o `Lock()` FCFS do Python, mas não totalmente garantido.

#### 3.3&nbsp;Conclusão da Fase 2  

Quebrar uma única condição necessária (espera circular) basta para impedir deadlocks, ressaltando a importância de um design cuidadoso em sistemas concorrentes.

---

### 4. Considerações Finais  

* Deadlocks ocorrem em SGBDs, sistemas de arquivos, APIs distribuídas.  
* Dominar processos, threads e mecanismos de sincronização é vital para software robusto.  
* O experimento ilustra que pequenas escolhas de projeto (ordem de aquisição) têm enorme impacto em confiabilidade e desempenho.

---

_Fim do Relatório_
