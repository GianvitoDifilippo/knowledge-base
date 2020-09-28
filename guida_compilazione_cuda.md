# Guida alla compilazione di ComputeAngle in Visual Studio 2019

Per prima cosa, se non presente, occorre installare il Toolkit di NVIDIA per CUDA, reperibile al seguente [link](https://developer.nvidia.com/cuda-downloads).

## Creazione e inizializzazione del progetto
Creare un nuovo progetto in Visual Studio e scegliere **CUDA x.x Runtime**, dove x.x indica la versione utilizzata, cercando "cuda" nella barra di ricerca.
Eliminare il file già presente chiamato "kernel.cu" e creare una cartella contenente il codice da compilare. Selezionare tutti gli elementi della cartella e includerli nel progetto. La lista dei files necessari alla compilazione del progetto è:
* `coder_array.h`
* `computeAngle.h` e `computeAngle.cu`
* `computeAngle_emxAPI.h` e `computeAngle_emxAPI.cu`
* `computeAngle_emxutil.h` e `computeAngle_emxutil.cu`
* `computeAngle_emxinitialize.h` e `computeAngle_emxinitialize.cu`
* `computeAngle_emxterminate.h` e `computeAngle_emxterminate.cu`
* `computeAngle_types.h`
* `rtwtypes.h` e `tmwtypes.h`

## main.cpp
Il codice di main.cpp è il seguente:
```c
#include <stdio.h>                // Per printf
#include "computeAngle.h"         // Per computeAngle
#include "computeAngle_emxAPI.h"  // Per emxInitArray_char_T and emxDestroyArray_char_T
#include "cuda_runtime.h"         // Per cudaMallocManaged and cudaFree

int main(int argc, char* argv[])
{
	// Dichiarazione variabili
	emxArray_char_T* hvsb;
	double* angle_rev;
	double angle = 65.1;

	// Allocazione e inizializzazione delle variabili
	cudaMallocManaged(&angle_rev, sizeof(double));
	emxInitArray_char_T(&hvsb, 2);

	// Funzione principale
	computeAngle(angle, hvsb, angle_rev);

	// Mostra input e output
	printf("Input: a = %f\n", angle);
	printf("Ran: angle_rev = %f\n", *angle_rev);
	printf("Ran: hvsb = %s\n", hvsb->data);

	// Libera la memoria
	emxDestroyArray_char_T(hvsb);
	cudaFree(angle_rev);

	return 0;
}
```
Ogni volta che bisogna allocare la memoria in CUDA, chiamare la funzione `cudaMallocManaged` anziché `malloc`: ciò renderà la memoria leggibile sia dalla CPU che dalla GPU; allo stesso modo, usare `cudaFree` al posto di `free` per liberare la memoria precedentemente allocata. La differenza tra `cudaMallocManaged` e `malloc` è che la prima funzione accetta un riferimento al puntatore di memoria dichiarato precedentemente, oltre che alla dimensione della memoria da allocare. Invece, per quanto riguarda emxArray_char_T, non bisogna allocare la memoria manualmente con `cudaMallocManaged`, bensì utilizzare la funzione `emxInitArray_char_T`: il numero di dimensioni dovrebbe essere 2, in base al codice presente in `computeArray.cu`; allo stesso modo, per liberare la memoria utilizzare `emxDestroyArray_char_T`.

## Cause di problemi
Assicurarsi che i file .cu siano correttamente interpretati dal compilatore come sorgente `CUDA C/C++`. Cliccare con il destro su di essi e nelle proprietà, accertarsi che il tipo di elemento sia impostato su `CUDA C/C++`.
