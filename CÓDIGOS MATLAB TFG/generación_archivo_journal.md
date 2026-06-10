texto para escribir



% Genera el journal (.jou) del barrido parametrico de Fluent a partir del Excel
% de design points (Name | Q1 | Q2 | Q3, en W/m^3). Cargar en Fluent standalone.

function gen_journal()

% CONFIGURACIÓN
N_CASES           = 343;      % n° de design points a generar (343 = 7^3)
N_ITER            = 100;      % iteraciones por caso (configuracion final)
SAVE_CASE_PER_RUN = false;    % true -> write-case-data por caso (ocupa GB)
BACKUP_EVERY      = 30;       % backup .dat cada N casos (0 = desactivado)
WARM_RESTART      = false;    % false -> cold restart (hyb-init + yes por caso)

% Entrada (Excel de design points) y salida (.jou)
XLSX_IN  = 'iso_clip_7_casos.xlsx';
SHEET_IN = 'iso_clip';        % columnas: Name | Q1 | Q2 | Q3   (W/m^3)
JOU_OUT  = 'batch_343_final.jou';

% Rutas en la MAQUINA REMOTA (donde corre Fluent)
CASE_BASE_REMOTE = 'D:/scratch/Christian_Taboada/SIMULACION_FINAL/CON LA PARTE DEL APOYO/simulacion_apoyo_final_files/dp0/FFF-27/Fluent/horno_base_convergido';
S2S_FILE_REMOTE  = 'D:/scratch/Christian_Taboada/SIMULACION_FINAL/CON LA PARTE DEL APOYO/simulacion_apoyo_final_files/dp0/FFF-27/Fluent/radiation_v3.s2s.h5';
OUT_DIR_REMOTE   = 'D:/scratch/Christian_Taboada/SIMULACION_FINAL/CON LA PARTE DEL APOYO/batch_output_343';
CSV_OUT_REMOTE   = [OUT_DIR_REMOTE '/resultados_343.csv'];

INPUT_PARAMS = {'q_res1_Wm3','q_res2_Wm3','q_res3_Wm3'};
REPORT_DEFS  = {'t_avg_belt','t_avg_belt_interior_horno','t_max_belt','t_min_belt'};

% 1.Leer y depurar los design points del Excel
T = readtable(XLSX_IN, 'Sheet', SHEET_IN, 'VariableNamingRule','preserve');
nm_all = string(T{:,1});
P_all  = T{:,2:4};
val    = (nm_all ~= "") & ~ismissing(nm_all) & all(~isnan(P_all),2);
nm_all = nm_all(val);   P_all = P_all(val,:);
nUse   = min(N_CASES, numel(nm_all));
Name   = nm_all(1:nUse);   Pv = P_all(1:nUse,:);

if WARM_RESTART
    restart_str = 'WARM (sin hyb-initialization)';
else
    restart_str = 'COLD (hyb-initialization por caso)';
end
if SAVE_CASE_PER_RUN, save_str = 'True'; else, save_str = 'False'; end
if BACKUP_EVERY > 0, backup_str = num2str(BACKUP_EVERY); else, backup_str = 'desactivado'; end

C = {};   % acumulador de lineas del journal

% Cabecera y preparación
C{end+1} = [';  ' JOU_OUT];
C{end+1} = ';  Generado automaticamente por gen_journal.m';
C{end+1} = [';  Casos: ' num2str(nUse) '   Iteraciones por caso: ' num2str(N_ITER)];
C{end+1} = [';  Restart: ' restart_str];
C{end+1} = [';  Guardar case+data por caso: ' save_str];
C{end+1} = [';  Backup cada N casos: ' backup_str];
C{end+1} = ';  PRE-FLIGHT CHECK (verificar MANUALMENTE antes de cargar el .jou)';
C{end+1} = ';    1.El directorio remoto:';
C{end+1} = [';         ' OUT_DIR_REMOTE];
C{end+1} = ';       DEBE existir. Si no existe, /file/start-transcript no';
C{end+1} = ';       puede crear el fichero, el transcript no arranca, y luego';
C{end+1} = ';       /file/stop-transcript falla con:';
C{end+1} = ';         "Error: A transcript has not been started."';
C{end+1} = ';       Ademas, write-results-csv tambien fallaria al escribir.';
C{end+1} = ';';
C{end+1} = ';    2.Fluent abierto en modo STANDALONE (NO desde Workbench).';
C{end+1} = ';       Workbench bloquea los comandos TUI con error workflow/wb.';
C{end+1} = '';
C{end+1} = '';
C{end+1} = '; 1.PREPARACION';
C{end+1} = '';
C{end+1} = '; 1.1.Leer case + data base (convergido, ventiladores 60%)';
C{end+1} = ['/file/read-case "' CASE_BASE_REMOTE '.cas.h5"'];
C{end+1} = ['/file/read-data "' CASE_BASE_REMOTE '.dat.h5"'];
C{end+1} = '';
C{end+1} = '; 1.2.Re-leer view factors S2S (geometricos, no cambian con los parametros)';
C{end+1} = '/define/models/radiation/s2s/read-existing-view-factors';
C{end+1} = ['"' S2S_FILE_REMOTE '"'];
C{end+1} = 'yes';
C{end+1} = '';
C{end+1} = '; 1.3.Verificacion: listar Named Expressions';
C{end+1} = '(display "\nNamed Expressions disponibles\n")';
C{end+1} = '/define/named-expressions/list';
C{end+1} = '(display "fin lista\n")';
C{end+1} = '';
C{end+1} = '; 1.4.Definiciones Scheme: acumulador de resultados y escritura CSV';
C{end+1} = '(define results ''())';
C{end+1} = '';
C{end+1} = '(define (dec->comma x)';
C{end+1} = '  (if (number? x)';
C{end+1} = '      (list->string';
C{end+1} = '        (map (lambda (c) (if (char=? c #\.) #\, c))';
C{end+1} = '             (string->list (number->string (exact->inexact x)))))';
C{end+1} = '      x))';
C{end+1} = '';
C{end+1} = '(define (write-results-csv path)';
C{end+1} = '  (with-output-to-file path';
C{end+1} = '    (lambda ()';
C{end+1} = '      (display "Name;Q1_Wm3;Q2_Wm3;Q3_Wm3;T_avg_belt_K;T_avg_interior_K;T_max_K;T_min_K")';
C{end+1} = '      (newline)';
C{end+1} = '      (for-each';
C{end+1} = '        (lambda (row)';
C{end+1} = '          (let loop ((r row) (first #t))';
C{end+1} = '            (cond ((null? r) (newline))';
C{end+1} = '                  (else';
C{end+1} = '                   (if (not first) (display ";"))';
C{end+1} = '                   (display (dec->comma (car r)))';
C{end+1} = '                   (loop (cdr r) #f)))))';
C{end+1} = '        (reverse results)))))';
C{end+1} = '';
C{end+1} = '; Helpers para extraccion de valores via transcript.';
C{end+1} = '; NOTA: %report-definition-eval NO existe en esta build de Fluent 2023 R2.';
C{end+1} = '; Enfoque: /file/start-transcript + /solve/report-definitions/compute + parseo.';
C{end+1} = '; Se usan integer->char para evitar problemas con literales de caracter Scheme.';
C{end+1} = '';
C{end+1} = '(define *chr-newline* (integer->char 10))';
C{end+1} = '(define *chr-return*  (integer->char 13))';
C{end+1} = '(define *chr-space*   (integer->char 32))';
C{end+1} = '';
C{end+1} = '(define (mi-read-line)';
C{end+1} = '  (let loop ((chars ''()))';
C{end+1} = '    (let ((c (read-char)))';
C{end+1} = '      (cond';
C{end+1} = '        ((eof-object? c)';
C{end+1} = '         (if (null? chars) c (list->string (reverse chars))))';
C{end+1} = '        ((char=? c *chr-newline*) (list->string (reverse chars)))';
C{end+1} = '        ((char=? c *chr-return*)  (loop chars))';
C{end+1} = '        (else (loop (cons c chars)))))))';
C{end+1} = '';
C{end+1} = '; Divide una cadena en tokens separados por espacio (devuelve lista de strings).';
C{end+1} = '(define (split-by-space s)';
C{end+1} = '  (let loop ((i 0) (start 0) (acc ''()))';
C{end+1} = '    (cond';
C{end+1} = '      ((= i (string-length s))';
C{end+1} = '       (reverse (if (> i start) (cons (substring s start i) acc) acc)))';
C{end+1} = '      ((char=? (string-ref s i) *chr-space*)';
C{end+1} = '       (loop (+ i 1) (+ i 1)';
C{end+1} = '             (if (> i start) (cons (substring s start i) acc) acc)))';
C{end+1} = '      (else (loop (+ i 1) start acc)))))';
C{end+1} = '';
C{end+1} = '; Recorre todos los tokens de la linea y devuelve el ULTIMO que sea numerico.';
C{end+1} = '; Tolera unidades pegadas al final (p.ej. ''... 752.34 [K]'') porque ''[K]'' no';
C{end+1} = '; convierte a numero y se ignora, quedandose con 752.34 como ultimo numerico.';
C{end+1} = '; Devuelve #f si la linea no contiene ningun numero.';
C{end+1} = '(define (ultimo-numero-de-linea linea)';
C{end+1} = '  (let loop ((toks (split-by-space linea)) (best #f))';
C{end+1} = '    (cond ((null? toks) best)';
C{end+1} = '          (else';
C{end+1} = '           (let ((n (string->number (car toks))))';
C{end+1} = '             (loop (cdr toks) (if (and n (number? n)) n best)))))))';
C{end+1} = '';
C{end+1} = '(define (safe-ref lst i default)';
C{end+1} = '  (cond ((null? lst) default)';
C{end+1} = '        ((= i 0) (car lst))';
C{end+1} = '        (else (safe-ref (cdr lst) (- i 1) default))))';
C{end+1} = '';
C{end+1} = '; Detecta si una linea contiene ''iso-clip-'' (prefijo de los nombres de superficie';
C{end+1} = '; de las report defs). Permite saltar la cabecera de sesion que Fluent imprime';
C{end+1} = '; en el PRIMER transcript de la sesion (Build Id, PIDs de procesos paralelos,';
C{end+1} = '; ano del Transcript Start Time). Esa cabecera contaminaba el parser con';
C{end+1} = '; numeros espurios y producia el caso 1 con valores absurdos (bug 2026-05-24).';
C{end+1} = '(define (contiene-iso-clip? linea)';
C{end+1} = '  (let* ((n (string-length linea))';
C{end+1} = '         (patron "iso-clip-")';
C{end+1} = '         (m (string-length patron)))';
C{end+1} = '    (let loop ((i 0))';
C{end+1} = '      (cond';
C{end+1} = '        ((> (+ i m) n) #f)';
C{end+1} = '        ((string=? (substring linea i (+ i m)) patron) #t)';
C{end+1} = '        (else (loop (+ i 1)))))))';
C{end+1} = '';
C{end+1} = '(define (parse-transcript fichero n)';
C{end+1} = '  (with-input-from-file fichero';
C{end+1} = '    (lambda ()';
C{end+1} = '      (let loop ((nums ''()))';
C{end+1} = '        (cond';
C{end+1} = '          ((= (length nums) n) (reverse nums))';
C{end+1} = '          (else';
C{end+1} = '           (let ((linea (mi-read-line)))';
C{end+1} = '             (cond';
C{end+1} = '               ((eof-object? linea)';
C{end+1} = '                (display (format #f "AVISO: solo ~a de ~a valores en transcript\n"';
C{end+1} = '                                 (length nums) n))';
C{end+1} = '                (reverse nums))';
C{end+1} = '               ((not (contiene-iso-clip? linea))';
C{end+1} = '                ; Salta cabecera de sesion, prompts, lineas en blanco y echos.';
C{end+1} = '                (loop nums))';
C{end+1} = '               (else';
C{end+1} = '                (let ((val (ultimo-numero-de-linea linea)))';
C{end+1} = '                  (if (and val (number? val))';
C{end+1} = '                      (loop (cons val nums))';
C{end+1} = '                      (loop nums))))))))))))';
C{end+1} = '';
C{end+1} = '';
C{end+1} = '; 2.CASOS';
C{end+1} = '';

% Bloque por caso
for i = 1:nUse
    nm = char(Name(i));
    s1 = sprintf('%.4f', Pv(i,1));
    s2 = sprintf('%.4f', Pv(i,2));
    s3 = sprintf('%.4f', Pv(i,3));
    tcaso = [OUT_DIR_REMOTE '/tmp_rd_values_' num2str(i) '.txt'];

    C{end+1} = ['; CASO ' num2str(i) ' / ' num2str(nUse) ': ' nm ' (' s1 ', ' s2 ', ' s3 ')'];
    C{end+1} = ['(display "\nCASO ' num2str(i) ' / ' num2str(nUse) '\n")'];
    C{end+1} = '';
    C{end+1} = ['/define/named-expressions/edit ' INPUT_PARAMS{1} ' definition "' s1 ' [W/m^3]" quit'];
    C{end+1} = ['/define/named-expressions/edit ' INPUT_PARAMS{2} ' definition "' s2 ' [W/m^3]" quit'];
    C{end+1} = ['/define/named-expressions/edit ' INPUT_PARAMS{3} ' definition "' s3 ' [W/m^3]" quit'];
    C{end+1} = '';
    if ~WARM_RESTART
        % hyb-initialization en Fluent 2023 R2 abre el prompt
        %   "Do you want to discard the data and proceed? [no]"
        % el 'yes' explicito de la linea siguiente fuerza el cold restart real.
        C{end+1} = '/solve/initialize/hyb-initialization';
        C{end+1} = 'yes';
        C{end+1} = '';
    end
    C{end+1} = ['/solve/iterate ' num2str(N_ITER)];
    C{end+1} = '';
    C{end+1} = '; Extraccion de valores via transcript (sin %report-definition-eval)';
    C{end+1} = '; Fichero unico por caso -> nunca pre-existe -> sin dialogo "OK to overwrite?"';
    C{end+1} = ['/file/start-transcript "' tcaso '"'];
    for k = 1:numel(REPORT_DEFS)
        C{end+1} = ['/solve/report-definitions/compute ' REPORT_DEFS{k}];
        C{end+1} = '';   % linea en blanco: cierra el prompt interactivo del compute
    end
    C{end+1} = '/file/stop-transcript';
    C{end+1} = '';
    C{end+1} = ['(let* ((name "' nm '")'];
    C{end+1} = ['       (p1 ' s1 ') (p2 ' s2 ') (p3 ' s3 ')'];
    C{end+1} = ['       (vals    (parse-transcript "' tcaso '" 4))'];
    C{end+1} = '       (tavg    (safe-ref vals 0 0.0))';
    C{end+1} = '       (tavgint (safe-ref vals 1 0.0))';
    C{end+1} = '       (tmax    (safe-ref vals 2 0.0))';
    C{end+1} = '       (tmin    (safe-ref vals 3 0.0)))';
    C{end+1} = '  (set! results (cons (list name p1 p2 p3 tavg tavgint tmax tmin) results))';
    C{end+1} = '  (display (format #f "  tavg=~a  tavgint=~a  tmax=~a  tmin=~a\n"';
    C{end+1} = '                   tavg tavgint tmax tmin))';
    C{end+1} = ['  (write-results-csv "' CSV_OUT_REMOTE '"))'];
    C{end+1} = '';
    if SAVE_CASE_PER_RUN
        C{end+1} = ['/file/write-case-data "' OUT_DIR_REMOTE '/caso_' num2str(i) '"'];
        C{end+1} = '';
    end
    if BACKUP_EVERY > 0 && mod(i, BACKUP_EVERY) == 0
        C{end+1} = ['/file/write-data "' OUT_DIR_REMOTE '/backup_cada_' num2str(BACKUP_EVERY) '_tras_caso_' num2str(i) '"'];
        C{end+1} = '';
    end
end

% Footer
C{end+1} = '';
C{end+1} = '; 3.FIN';
C{end+1} = '(display "\nBATCH COMPLETADO\n")';
C{end+1} = '';

% Escritura del .jou con saltos de linea LF
% Se usa fwrite (no fprintf) para escribir los bytes literalmente:
txt = strjoin(C, char(10));
fid = fopen(JOU_OUT, 'w');
fwrite(fid, txt, 'char');
fclose(fid);
fprintf('Journal generado: %s  (%d casos, %d lineas)\n', JOU_OUT, nUse, numel(C));
end
