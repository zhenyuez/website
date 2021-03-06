#Measure the properties of an MPS wavefunction#

Here we will learn how to measure some properties of the
Heisenberg model wavefunction by adding to the code from the [[previous example|recipes/basic_dmrg]].
Essentially we need three pieces to take an expectation value of some property: the operator 
corresponding to that property, the wavefunction, and a way to do the inner
product of the operator with the wavefunction.

For example, consider trying to measure the z-component of
the spin on some site `j` in the Heisenberg chain. This operator is a method of the `SpinOne model(N);` class
that we defined last time, and can be represented by an ITensor:

<code>
ITensor szj_op = model.sz(j);
</code>

Likewise there are operators for the site `j` spin raising (S+) and lowering operators (S-),
methods `sp(j)` and `sm(j)`, respectively.
Now there is an efficient way to take the inner product of these local operators with the MPS
wavefunction. To do this, we must first call the function

<code>
psi.position(j);
</code>

_This step is absolutely vital_.  It tells the MPS wavefunction to put all the information
about the amplitudes of the various spin occupations into the site at `j`.
Without this step, we would not be measuring `sz(j)` in an orthonormal environment and would
get the wrong answer.  Getting the needed amplitudes is very simple, 
and they will form the ket part of our Dirac "bra-ket" inner product:

<code>
ITensor ket = psi.AA(j);
</code>

After calling `psi.position(j)`, all the `psi.AA(j+n)` tensors for `n != 0` form
the basis weighted by the wavefunction amplitudes.
As basis tensors, these `psi.AA(j+n)` do not have amplitude information; only the `psi.AA(j)` does. 
That is why it is vital to call `psi.position(j)` before doing a measurement at site `j`.

To get the bra part of the Dirac bra-ket, think about the ket as a column vector, the operator
as a matrix, and the bra as a row vector.  We get the bra by turning the ket into a row vector and conjugating
any imaginary parts.  The way we do this is simple:

<code>
ITensor bra = conj(primesite(ket));
</code>

The `primesite` function is what turns the ket into a row vector (because it will contract with the 
row index of our operator), and `conj` does the conjugation.
Now we are ready to measure the expectation value of `sz(j)` by using the inner product function `Dot`:

<code>
Real szj = Dot(bra, szj_op*ket);
</code> 

Let's do another, more complicated measurement that will require our kets to be representing information
on two sites `j` and `j+1`.  The vector product S(j) dot S(j+1) will do just fine.  In terms of the 
z-component and raising and lowering operators, S(j) dot S(j+1) = `sz(j)*sz(j+1) + 0.5*( sp(j)*sm(j+1) + sp(j+1)*sm(j) )`.
For example, one of the operators we'll need is

<code>
ITensor spm_op = model.sp(j)*model.sm(j+1);
</code>

To represent the wavefunction for two sites, we simply call the `bondTensor` method of the wavefunction:

<code>
ITensor bondket = psi.bondTensor(j); 
</code>

The method `psi.bondTensor(j)` is analogous to `psi.AA(j)`, except that 
`bondTensor` encodes bond information between `j` and `j+1`, so long as `psi.position(j)` has been called previously.
The `bondbra` is made the same as the `bra` from earlier:

<code>
ITensor bondbra = conj(primesite(bondket));
</code>

And expectation values are realized in the same way:

<code>
Real spm = 0.5\*Dot(bondbra, spm_op\*bondket);
</code>

Below is a complete code for measuring properties of the MPS wavefunction.  
The code writes to file a list of Sz(j) and S(j) dot S(j+1) for plotting.


<code>
\#include "core.h"
\#include "model/spinone.h"
\#include "hams/heisenberg.h"
using boost::format;
using namespace std;

int main(int argc, char\* argv[])
{
    int N = 100;
    int nsweep = 5;
    int minm = 1;
    int maxm = 100;
    Real cutoff = 1E-5;

    SpinOne model(N);

    MPO H = Heisenberg(model);

    InitState initState(N);
    for(int i = 1; i <= N; ++i) 
        initState(i) = (i%2==1 ? model.Up(i) : model.Dn(i));

    MPS psi(model,initState);

    cout << format("Initial energy = %.5f\n")%psiHphi(psi,H,psi);

    Sweeps sweeps(Sweeps::ramp\_m,nsweep,minm,maxm,cutoff);
    Real En = dmrg(psi,H,sweeps);

    cout << format("\nGround State Energy = %.10f\n")%En;

    //
    //MEASURING SPIN
    //

    //vector of z-components of spin for each site
    Vector Sz(N);
    Sz = 0.0;
    std::ofstream szf("Sz"); //file to write Szj info to

    //Sj dot Sj+1 by means of Splus and Sminus
    Vector SdotS(N-1); 
    SdotS = 0.0;

    //file to write SdotS information to
    std::ofstream sdots("SdotS"); 

    //also sum up the SdotS and see if it is 
    //equal to our ground state energy
    Real sumSdotS = 0.0;

    for(int j=1; j<=N; j++) {

        //move psi to get ready to measure at position j
        psi.position(j);
        //after calling psi.position(j), psi.AA(j) returns a 
        //local representation of the wavefunction,
        //which is the proper way to make measurements / take 
        //expectation values with local operators.

        //Dirac "ket" for wavefunction
        ITensor ket = psi.AA(j);
        //Dirac "bra" for wavefunction
        ITensor bra = conj(primesite(ket));

        //operator for sz at site j
        ITensor szj\_op = model.sz(j);

         //take an inner product 
        Sz(j) = Dot(bra, szj\_op\*ket);
        szf << Sz(j) << endl; //print to file

        if (j<N) { 

            //make a bond ket/bra based on wavefunction 
            //representing sites j and j+1:
            //psi.bondTensor(j) is analogous to psi.AA(j), 
            //except that bondTensor encodes bond information 
            //between j and j+1, so long as 
            //psi.position(j) has been called
            ITensor bondket = psi.bondTensor(j); 
            ITensor bondbra = conj(primesite(bondket)); 

            ITensor szz\_op = model.sz(j)\*model.sz(j+1); 
            //start with z components
            SdotS(j) = Dot(bondbra, szz\_op\*bondket);

            //add in S+ and S- components:
            //use sp(j)\*sm(j) and its conjugate, 
            //also note the one-half out front:
            ITensor spm\_op = model.sp(j)\*model.sm(j+1);
            SdotS(j) += 0.5\*Dot(bondbra, spm\_op\*bondket);
            ITensor smp\_op = model.sm(j)\*model.sp(j+1);
            SdotS(j) += 0.5\*Dot(bondbra, smp\_op\*bondket);

            sdots << SdotS(j) << endl; //print to file

            //figure out the sum of SdotS.  
            //should be the same as the Heisenberg energy
            sumSdotS += SdotS(j); 
        }
    }

    cout << format("\nSum of SdotS is %f\n")%sumSdotS;

    return 0;
}


</code>

<br>
[[Back to Recipes|recipes]]
