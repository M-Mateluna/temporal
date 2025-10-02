(function () {
  const $ = (s) => document.querySelector(s);
  const $$ = (s) => Array.prototype.slice.call(document.querySelectorAll(s));

  const form = $('#formVoto');
  const selRegion = $('#idregion');
  const selComuna = $('#idcomuna');
  const selCandidato = $('#idcandidato');
  const wrapFuentes = $('#fuentesWrap');

  // ---------- Helpers ----------
  function alertMsg(m) { alert(m); }
  function isEmail(v) { return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v); }
  function hasLettersAndNumbers(v) {
    return /[A-Za-z]/.test(v) && /\d/.test(v) && /^[A-Za-z0-9]+$/.test(v);
  }
  function normalizeRut(rut) {
    rut = (rut || '').replace(/\./g,'').replace(/\s+/g,'').toUpperCase();
    if (rut.indexOf('-') === -1 && rut.length > 1) {
      rut = rut.slice(0, -1) + '-' + rut.slice(-1);
    }
    return rut;
  }
  function validarRut(rut) {
    rut = normalizeRut(rut);
    const parts = rut.split('-');
    if (parts.length !== 2) return false;
    const num = parts[0].replace(/\D/g,'');
    const dv  = parts[1];
    if (!num || !dv) return false;

    let suma = 0, factor = 2;
    for (let i = num.length - 1; i >= 0; i--) {
      suma += parseInt(num.charAt(i),10) * factor;
      factor = (factor === 7) ? 2 : factor + 1;
    }
    const resto = suma % 11;
    const calc  = 11 - resto;
    const calcDV = (calc === 11) ? '0' : (calc === 10 ? 'K' : String(calc));
    return calcDV === dv.toUpperCase();
  }

  // ---------- AJAX ----------
  function getJSON(url) {
    return fetch(url, { credentials: 'same-origin' }).then(r => r.json());
  }

  // Construye application/x-www-form-urlencoded,
  // soporta correctamente campos múltiples con [].
  function postForm(url, data) {
    const body = new URLSearchParams();
    Object.keys(data).forEach(k => {
      const v = data[k];
      if (Array.isArray(v)) {
        // Para arreglos, enviar como idfuentes[] (clave con corchetes)
        v.forEach(item => body.append(k + '[]', item));
      } else {
        body.append(k, v);
      }
    });
    return fetch(url, {
      method: 'POST',
      headers: { 'Content-Type':'application/x-www-form-urlencoded;charset=UTF-8' },
      credentials: 'same-origin',
      body
    }).then(r => r.json());
  }

  // ---------- Cargas iniciales (usa tus funciones vía command) ----------
  function cargarRegiones() {
    getJSON('./command?accion=listar_regiones').then(res => {
      if (!res.ok) return alertMsg(res.msg || 'Error cargando regiones');
      selRegion.innerHTML = '<option value="">Seleccione...</option>' +
        res.data.map(r => `<option value="${r.id}">${r.nombre}</option>`).join('');
    });
  }

  function cargarComunas(idregion) {
    selComuna.disabled = true;
    selComuna.innerHTML = '<option value="">Seleccione...</option>';
    if (!idregion) return;
    getJSON('./command?accion=listar_comunas&idregion=' + encodeURIComponent(idregion))
      .then(res => {
        if (!res.ok) return alertMsg(res.msg || 'Error cargando comunas');
        selComuna.innerHTML = '<option value="">Seleccione...</option>' +
          res.data.map(c => `<option value="${c.id}">${c.nombre}</option>`).join('');
        selComuna.disabled = false;
      });
  }

  function cargarCandidatos() {
    getJSON('./command?accion=listar_candidatos').then(res => {
      if (!res.ok) return alertMsg(res.msg || 'Error cargando candidatos');
      selCandidato.innerHTML = '<option value="">Seleccione...</option>' +
        res.data.map(c => `<option value="${c.id}">${c.nombre}</option>`).join('');
    });
  }

  function cargarFuentes() {
    getJSON('./command?accion=listar_fuentes').then(res => {
      if (!res.ok) return alertMsg(res.msg || 'Error cargando fuentes');
      wrapFuentes.innerHTML = res.data.map(f => `
        <label><input type="checkbox" name="idfuentes[]" value="${f.id}"> ${f.nombre}</label>
      `).join('');
    });
  }

  selRegion.addEventListener('change', function() {
    cargarComunas(this.value);
  });

  // ---------- Envío del formulario ----------
  form.addEventListener('submit', function(e) {
    e.preventDefault();

    const nombreapellido = $('#nombreapellido').value.trim();
    const alias          = $('#alias').value.trim();
    const rut            = normalizeRut($('#rut').value.trim());
    const email          = $('#email').value.trim();
    const idregion       = selRegion.value;
    const idcomuna       = selComuna.value;
    const idcandidato    = selCandidato.value;
    const fuentes        = $$('#fuentesWrap input[type="checkbox"]:checked').map(ch => ch.value);

    // Validaciones (coherentes con lo que valida la función SQL)
    if (!nombreapellido) return alertMsg('El campo "Nombre y Apellido" es obligatorio');
    if (!(alias && alias.length > 5 && hasLettersAndNumbers(alias)))
      return alertMsg('Alias inválido: mínimo 6 caracteres, alfanumérico con letras y números');
    if (!rut) return alertMsg('El campo RUT es obligatorio');
    if (!validarRut(rut)) return alertMsg('RUT inválido');
    if (!email) return alertMsg('El campo Email es obligatorio');
    if (!isEmail(email)) return alertMsg('Email inválido');
    if (!idregion) return alertMsg('Debe seleccionar una Región');
    if (!idcomuna) return alertMsg('Debe seleccionar una Comuna');
    if (!idcandidato) return alertMsg('Debe seleccionar un Candidato');
    if (fuentes.length < 2) return alertMsg('Debe elegir al menos dos opciones en "¿Cómo se enteró de nosotros?"');

    const payload = {
      nombreapellido,
      alias,
      rut,
      email,
      idregion,
      idcomuna,
      idcandidato,
      // clave con [] para que PHP la reciba como $_POST['idfuentes'] (array)
      'idfuentes': fuentes
    };

    postForm('./command?accion=registrar_voto', payload)
      .then(res => {
        if (!res.ok) return alertMsg(res.msg || 'No fue posible registrar el voto');
        alertMsg('¡Voto registrado correctamente! ID: ' + (res.data && res.data.idvoto ? res.data.idvoto : ''));
        form.reset();
        selComuna.disabled = true;
      })
      .catch(() => alertMsg('Error de red'));
  });

  // ---------- init ----------
  cargarRegiones();
  cargarCandidatos();
  cargarFuentes();
})();
