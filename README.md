from flask import Flask, request, render_template_string, redirect, url_for
import os
import importlib.util

# Carica dinamicamente il modulo `python compa_team.py` (il file contiene uno spazio nel nome)
PROJECT_DIR = os.path.dirname(__file__)
MODULE_PATH = os.path.join(PROJECT_DIR, "python compa_team.py")

if not os.path.exists(MODULE_PATH):
    raise FileNotFoundError(f"Impossibile trovare il modulo: {MODULE_PATH}")

spec = importlib.util.spec_from_file_location("compa_module", MODULE_PATH)
compa_module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(compa_module)

# Recupera la funzione calcola_indice_forza dal modulo caricato
if not hasattr(compa_module, "calcola_indice_forza"):
    raise AttributeError("Il modulo caricato non contiene 'calcola_indice_forza'")

calcola_indice_forza = compa_module.calcola_indice_forza

app = Flask(__name__)

FORM_HTML = """
<!doctype html>
<title>CompaTeam - Web</title>
<h1>CompaTeam: Confronto Forze Squadre</h1>
<form method="post" action="/compute">
  <fieldset>
    <legend>Squadra A (Casa)</legend>
    Nome: <input name="nome_a" required><br>
    Attacco: <input name="attacco_a" type="number" min="0" max="10" required><br>
    Difesa: <input name="difesa_a" type="number" min="0" max="10" required><br>
  </fieldset>
  <fieldset>
    <legend>Squadra B (Ospite)</legend>
    Nome: <input name="nome_b" required><br>
    Attacco: <input name="attacco_b" type="number" min="0" max="10" required><br>
    Difesa: <input name="difesa_b" type="number" min="0" max="10" required><br>
  </fieldset>
  <p><button type="submit">Calcola</button></p>
</form>
"""

RESULT_HTML = """
<!doctype html>
<title>Risultato - CompaTeam</title>
<h1>Risultato CompaTeam</h1>
<p><strong>{{ nome_a }}</strong> (Casa) - Indice: {{ forza_a }}</p>
<p><strong>{{ nome_b }}</strong> (Ospite) - Indice: {{ forza_b }}</p>
<p><em>Differenziale: {{ differenziale }}</em></p>
<h2>Previsione</h2>
<p>{{ previsione }}</p>
<p><a href="{{ url_for('index') }}">Nuova comparazione</a></p>
"""


@app.route("/", methods=["GET"])
def index():
    return render_template_string(FORM_HTML)


@app.route("/compute", methods=["POST"])
def compute():
    try:
        nome_a = request.form.get("nome_a", "").strip()
        nome_b = request.form.get("nome_b", "").strip()
        attacco_a = int(request.form.get("attacco_a", 0))
        difesa_a = int(request.form.get("difesa_a", 0))
        attacco_b = int(request.form.get("attacco_b", 0))
        difesa_b = int(request.form.get("difesa_b", 0))
    except (ValueError, TypeError):
        return redirect(url_for('index'))

    forza_a = calcola_indice_forza(nome_a, attacco_a, difesa_a, True)
    forza_b = calcola_indice_forza(nome_b, attacco_b, difesa_b, False)

    differenziale = round(forza_a - forza_b, 2)

    if differenziale >= 1.5:
        previsione = f"Vittoria Schiacciante per {nome_a} (Differenziale: +{differenziale:.2f})"
    elif differenziale >= 0.5:
        previsione = f"Vittoria Probabile per {nome_a} (Differenziale: +{differenziale:.2f})"
    elif differenziale >= -0.5:
        previsione = f"Partita Equilibrata (Differenziale: {differenziale:.2f}) - Pareggio o Vittoria Minima."
    elif differenziale >= -1.5:
        previsione = f"Vittoria Probabile per {nome_b} (Differenziale: {differenziale:.2f})"
    else:
        previsione = f"Vittoria Schiacciante per {nome_b} (Differenziale: {differenziale:.2f})"

    return render_template_string(RESULT_HTML,
                                  nome_a=nome_a,
                                  nome_b=nome_b,
                                  forza_a=f"{forza_a:.2f}",
                                  forza_b=f"{forza_b:.2f}",
                                  differenziale=f"{differenziale:.2f}",
                                  previsione=previsione)


if __name__ == '__main__':
    port = int(os.environ.get("PORT", 5000))
    app.run(host='0.0.0.0', port=port, debug=False)
