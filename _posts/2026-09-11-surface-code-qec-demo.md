---
layout: post
title:  "Simulating a Surface Code QEC Cycle with Qiskit"
date:   2026-09-11 12:00:00
description: An interactive-style walkthrough of a single quantum error correction cycle on a 3×3 surface code, embedded from a runnable Jupyter notebook
tags: qec qiskit superconducting
categories: reading
---

Quantum error correction (QEC) is the key ingredient that turns noisy physical qubits into reliable logical qubits. The **surface code** is currently the leading QEC scheme for superconducting processors: it arranges data qubits on a 2D lattice interleaved with ancilla qubits that repeatedly measure *stabilizers* — parity checks that reveal errors without disturbing the encoded information.

Rather than only describing the idea, I embedded a runnable Jupyter notebook below. It builds a single QEC cycle of the **3×3 surface code** (9 data qubits, 4 Z-type ancillas, 4 X-type ancillas) in Qiskit — including a unitary encoding step that prepares the logical $\lvert 0\rangle_L$ state — injects a deliberate error, and reads out the syndrome. Feel free to [download the notebook]({% link assets/ipynb/surface_code_demo_revamp.ipynb %}) and run it yourself — the required packages are `qiskit`, `qiskit-aer`, and `pylatexenc`.

<div class="jupyter-notebook" style="position: relative; width: 100%; margin: 0 auto;">
  <div class="jupyter-notebook-iframe-container">
    <iframe src="{{ '/assets/ipynb/surface_code_demo_revamp.ipynb.html' | relative_url }}" style="position: absolute; top: 0; left: 0; border-style: none;" width="100%" height="100%" onload="this.parentElement.style.paddingBottom = (this.contentWindow.document.documentElement.scrollHeight + 10) + 'px'"></iframe>
  </div>
</div>

## Try it yourself: syndrome lookup

The notebook is fully deterministic, so here is an interactive replica that runs entirely in your browser. Pick a data qubit and an error type; the widget computes the resulting 8-bit syndrome ($Z_1 Z_2 Z_3 Z_4 X_1 X_2 X_3 X_4$) and shows which stabilizers are violated. These results match the simulations above exactly.

<div id="syndrome-widget"></div>
<script>
(function () {
  var STABILIZERS = {
    Z: [{ name: 'Z1', data: [1, 4] }, { name: 'Z2', data: [4, 5, 7, 8] }, { name: 'Z3', data: [2, 3, 5, 6] }, { name: 'Z4', data: [6, 9] }],
    X: [{ name: 'X1', data: [2, 3] }, { name: 'X2', data: [1, 2, 4, 5] }, { name: 'X3', data: [5, 6, 8, 9] }, { name: 'X4', data: [7, 8] }]
  };
  var syndromes = {
    D1: { X: '10000000', Z: '00000100' }, D2: { X: '00100000', Z: '00001100' },
    D3: { X: '00100000', Z: '00001000' }, D4: { X: '11000000', Z: '00000100' },
    D5: { X: '01100000', Z: '00000110' }, D6: { X: '00110000', Z: '00000010' },
    D7: { X: '01000000', Z: '00000001' }, D8: { X: '01000000', Z: '00000011' },
    D9: { X: '00010000', Z: '00000010' }
  };
  var root = document.getElementById('syndrome-widget');
  root.innerHTML =
    '<div class="syndrome-controls">' +
      '<label>Data qubit <select id="syndrome-qubit">' +
        ['D1','D2','D3','D4','D5','D6','D7','D8','D9'].map(function (q) { return '<option' + (q === 'D5' ? ' selected' : '') + '>' + q + '</option>'; }).join('') +
      '</select></label> ' +
      '<label>Error type <select id="syndrome-type"><option>X</option><option>Z</option></select></label>' +
    '</div>' +
    '<div class="syndrome-bits" id="syndrome-bits"></div>' +
    '<p class="syndrome-note" id="syndrome-note"></p>';

  function render() {
    var q = document.getElementById('syndrome-qubit').value;
    var t = document.getElementById('syndrome-type').value;
    var bits = syndromes[q][t];
    var all = STABILIZERS.Z.concat(STABILIZERS.X);
    var holder = document.getElementById('syndrome-bits');
    holder.innerHTML = '';
    all.forEach(function (s, i) {
      var flip = bits[i] === '1';
      var cell = document.createElement('span');
      cell.className = 'syndrome-bit' + (flip ? ' flipped' : '');
      cell.textContent = s.name + ' ' + bits[i];
      holder.appendChild(cell);
    });
    var note = document.getElementById('syndrome-note');
    var zFlips = bits.slice(0, 4).split('').filter(function (b) { return b === '1'; }).length;
    var xFlips = bits.slice(4).split('').filter(function (b) { return b === '1'; }).length;
    if (bits === '00000000') {
      note.textContent = 'Clean syndrome — no stabilizer violated.';
    } else if (t === 'X') {
      note.textContent = 'An X error on ' + q + ' anticommutes with the Z-type stabilizers it touches, so ' + zFlips + ' Z-stabilizer bit(s) fire.';
    } else {
      note.textContent = 'A Z error on ' + q + ' anticommutes with the X-type stabilizers it touches, so ' + xFlips + ' X-stabilizer bit(s) fire.';
    }
  }
  document.getElementById('syndrome-qubit').addEventListener('change', render);
  document.getElementById('syndrome-type').addEventListener('change', render);
  render();
})();
</script>

<style>
  .syndrome-controls { margin: 1rem 0; display: flex; gap: 1.5rem; flex-wrap: wrap; }
  .syndrome-controls label { font-weight: 600; }
  .syndrome-controls select { margin-left: 0.5rem; padding: 0.2rem 0.4rem; }
  .syndrome-bits { display: flex; flex-wrap: wrap; gap: 0.5rem; margin: 0.75rem 0; }
  .syndrome-bit {
    display: inline-block; padding: 0.35rem 0.7rem; border-radius: 6px;
    font-family: var(--global-serif, monospace); font-size: 0.9rem;
    background: var(--global-bg, #f5f5f5); border: 1px solid var(--global-divider, #ddd);
  }
  .syndrome-bit.flipped {
    background: #b00020; color: #fff; border-color: #b00020; font-weight: 700;
  }
  .syndrome-note { font-size: 0.9rem; color: var(--global-text-secondary, #666); }
</style>

## What to look for

- **No error**: the syndrome is clean — all stabilizer parities read +1 (`00000000` modulo measurement noise from the ancilla preparation scheme used in the demo).
- **X error on a data qubit**: flips the surrounding **Z stabilizers**, so the first four syndrome bits light up.
- **Z error on a data qubit**: flips the surrounding **X stabilizers**, lighting up the last four syndrome bits.
- A data qubit in the middle of the lattice (D5) is adjacent to all four stabilizers of each type, so a single error there produces a maximal-weight syndrome — this is exactly the kind of pattern the decoder must disambiguate in real experiments.

Decoding these patterns back to the most likely physical error is the job of a classical decoder running in real time — the subject of the landmark experiment cited below.

# References

[1] S. Krinner et al, Realizing repeated quantum error correction in a distance-three surface code, Nature 605, 669–674 (2022). [doi:10.1038/s41586-022-04566-1](https://doi.org/10.1038/s41586-022-04566-1) \\
