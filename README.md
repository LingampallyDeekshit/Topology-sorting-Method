#include <stdio.h>
#include <stdlib.h>
#include <string.h>


int *queue_arr;
int qhead, qtail;

void qinit(int n) {
    queue_arr = (int*)malloc(sizeof(int)*n);
    qhead = qtail = 0;
}
void qpush(int x) { queue_arr[qtail++] = x; }
int qempty() { return qhead == qtail; }
int qpop() { return queue_arr[qhead++]; }
void qfree() { free(queue_arr); }

/* Build adjacency matrix from AL and RQ matrices */
void build_graph(int **adj, int P, int R, int **AL, int **RQ) {
    int N = P + R;
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            adj[i][j] = 0;

    for (int p = 0; p < P; ++p) {
        for (int r = 0; r < R; ++r) {
            int resNode = P + r;
            if (RQ[p][r]) {
                // process -> resource (request)
                adj[p][resNode] = 1;
            }
            if (AL[p][r]) {
                // allocation resource -> process
                adj[resNode][p] = 1;
            }
        }
    }
}

/* Kahn's topological sort.
   Returns number of processed nodes in *procCount.
   Fills processed[] (1 if popped/processed).
   Also fills order[] with processed node indices in order. */
void kahn_toposort(int **adj, int N, int *processed, int *order, int *procCount) {
    int *indeg = (int*)malloc(sizeof(int)*N);
    for (int i = 0; i < N; ++i) indeg[i] = 0;

    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            if (adj[i][j]) indeg[j]++;

    qinit(N);
    for (int i = 0; i < N; ++i)
        if (indeg[i] == 0) qpush(i);

    int cnt = 0;
    while (!qempty()) {
        int u = qpop();
        processed[u] = 1;
        order[cnt++] = u;
        for (int v = 0; v < N; ++v) {
            if (adj[u][v]) {
                indeg[v]--;
                if (indeg[v] == 0) qpush(v);
            }
        }
    }
    *procCount = cnt;
    qfree();
    free(indeg);
}

/* Check deadlock and return 1 if deadlock (cycle) exists, else 0.
   Also outputs safe order if no deadlock. */
int detect_deadlock_and_get_order(int **adj, int N, int *safeOrder, int *safeCount, int *inCycleMask) {
    for (int i = 0; i < N; ++i) inCycleMask[i] = 0;
    for (int i = 0; i < N; ++i) safeOrder[i] = -1;
    for (int i = 0; i < N; ++i) inCycleMask[i] = 0;

    int *processed = (int*)calloc(N, sizeof(int));
    int procCount = 0;
    kahn_toposort(adj, N, processed, safeOrder, &procCount);

    if (procCount == N) {
        *safeCount = procCount;
        free(processed);
        return 0; // no cycle
    } else {
        // mark nodes not processed -> part of cycle(s)
        for (int i = 0; i < N; ++i) if (!processed[i]) inCycleMask[i] = 1;
        free(processed);
        return 1; // deadlock detected
    }
}

int main() {
    int P, R;
    printf("Enter number of processes (P) and resources (R): ");
    if (scanf("%d %d", &P, &R) != 2) return 0;

    // allocate AL[P][R] and RQ[P][R]
    int **AL = (int**)malloc(sizeof(int*)*P);
    int **RQ = (int**)malloc(sizeof(int*)*P);
    for (int i = 0; i < P; ++i) {
        AL[i] = (int*)calloc(R, sizeof(int));
        RQ[i] = (int*)calloc(R, sizeof(int));
    }

    printf("Enter allocation matrix AL (P rows, R columns). AL[i][j]=1 if resource j allocated to process i:\n");
    for (int i = 0; i < P; ++i) for (int j = 0; j < R; ++j) scanf("%d", &AL[i][j]);

    printf("Enter request matrix RQ (P rows, R columns). RQ[i][j]=1 if process i requests resource j:\n");
    for (int i = 0; i < P; ++i) for (int j = 0; j < R; ++j) scanf("%d", &RQ[i][j]);

    int N = P + R;
    // adjacency matrix
    int **adj = (int**)malloc(sizeof(int*)*N);
    for (int i = 0; i < N; ++i) {
        adj[i] = (int*)calloc(N, sizeof(int));
    }

    build_graph(adj, P, R, AL, RQ);

    int *safeOrder = (int*)malloc(sizeof(int)*N);
    int safeCount = 0;
    int *inCycleMask = (int*)malloc(sizeof(int)*N);

    // resolution loop: if deadlock -> abort one process from cycle, free its allocated resources, repeat
    int *aborted = (int*)calloc(P, sizeof(int));
    int abortedCount = 0;

    while (1) {
        int dead = detect_deadlock_and_get_order(adj, N, safeOrder, &safeCount, inCycleMask);
        if (!dead) {
            printf("\nNo deadlock detected.\n");
            // print safe sequence of processes (filter only process nodes)
            printf("Safe sequence (nodes shown as P0..P%d and R0..R%d):\n", P-1, R-1);
            for (int i = 0; i < safeCount; ++i) {
                int node = safeOrder[i];
                if (node < P) printf("P%d ", node);
                else printf("R%d ", node - P);
            }
            printf("\n");
            break;
        } else {
            // deadlock exists. find a process node that is part of cycle and not yet aborted
            int victim = -1;
            for (int p = 0; p < P; ++p) {
                if (inCycleMask[p] && !aborted[p]) { victim = p; break; }
            }
            if (victim == -1) {
                // (rare) if no process found among cycle nodes, abort any un-aborted process
                for (int p = 0; p < P; ++p) if (!aborted[p]) { victim = p; break; }
            }
            if (victim == -1) {
                printf("No candidate to abort. Unable to resolve.\n");
                break;
            }
            // abort victim: free its allocated resources and clear its requests
            aborted[victim] = 1;
            abortedCount++;
            printf("\nDeadlock detected. Aborting process P%d to resolve deadlock.\n", victim);

            for (int r = 0; r < R; ++r) {
                if (AL[victim][r]) {
                    // free resource r
                    AL[victim][r] = 0;
                    // resource node had edge to this process; removing allocation will be done when rebuilding
                }
                // remove its outstanding requests too
                if (RQ[victim][r]) RQ[victim][r] = 0;
            }

            // Rebuild adjacency matrix from updated AL and RQ
            build_graph(adj, P, R, AL, RQ);

            // continue loop until no cycle
        }
    }

    if (abortedCount > 0) {
        printf("\nAborted processes to resolve deadlock:");
        for (int i = 0; i < P; ++i) if (aborted[i]) printf(" P%d", i);
        printf("\n");
    }

    // free memory
    for (int i = 0; i < P; ++i) { free(AL[i]); free(RQ[i]); }
    free(AL); free(RQ);
    for (int i = 0; i < N; ++i) free(adj[i]);
    free(adj);
    free(safeOrder); free(inCycleMask); free(aborted);

    return 0;
}
