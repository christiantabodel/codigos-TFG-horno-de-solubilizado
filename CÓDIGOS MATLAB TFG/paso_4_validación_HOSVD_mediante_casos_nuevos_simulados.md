Paso 4: validación del HOSVD mediante casos nuevos simulados.

Último paso correspondiente a la validación final del modelo. Se construye el HOSVD con la base de datos completa de los 343 casos, reteniendo cuatro modos por dirección, y se evalúa sobre 20 escenarios nuevos simulados de forma independiente en CFD, que se leen de una plantilla externa junto con sus resultados. Los casos se clasifican de manera automática en dentro o fuera del rango paramétrico estudiado, distinguiendo así la interpolación de la extrapolación, y para cada grupo se calculan el RMSE, el error máximo y el R², además del detalle caso a caso. El script mide también el tiempo de predicción por caso, construyendo el modelo una sola vez y promediando varias repeticiones, dato que se emplea en la comparativa de coste computacional frente a la simulación CFD. Genera siete figuras, de las que proceden las Figuras 53 a 56 de la memoria: predicción frente a CFD para ambos grupos, RMSE y error máximo por salida (dentro frente a fuera de rango), error absoluto de T_max caso a caso y distribución de los 20 casos en el espacio de potencias, con el rango estudiado delimitado. Los resultados se guardan en resultados_validacion_nuevos.mat.


```matlab
% Paso 4: validación del HOSVD con casos nuevos simulados aparte (dentro y fuera
% del rango estudiado) y medida del tiempo de predicción por caso.

clear; clc; close all;

% 0.CONFIGURACIÓN
nombre_mat  = 'datos_preparados.mat';
nombre_test = 'plantilla de prueba final HOSVD.xlsx';
r = 4; % modos retenidos en el HOSVD

% 1.CARGA DE DATOS
carpeta = fileparts(mfilename('fullpath'));
if isempty(carpeta); carpeta = pwd; end
if exist(fullfile(carpeta,nombre_mat),'file') ~= 2
    error('Falta %s. Ejecuta el paso 1 antes.', nombre_mat);
end
if exist(fullfile(carpeta,nombre_test),'file') ~= 2
    error('Falta "%s" (casos nuevos con resultados CFD).', nombre_test);
end
S = load(fullfile(carpeta,nombre_mat));
tensores = S.tensores;  col_salidas = S.col_salidas;
q1 = S.q1_niveles(:);  q2 = S.q2_niveles(:);  q3 = S.q3_niveles(:);
n_sal = numel(col_salidas);

Te = readtable(fullfile(carpeta,nombre_test));
Qe = Te{:, {'Q1_Wm3','Q2_Wm3','Q3_Wm3'}};
n_test = size(Qe,1);

% 2.CLASIFICACIÓN DENTRO / FUERA DE RANGO
Qmin = [min(q1) min(q2) min(q3)];  Qmax = [max(q1) max(q2) max(q3)];
fuera  = any(Qe < Qmin | Qe > Qmax, 2);
dentro = ~fuera;
fprintf('Casos nuevos: %d  (dentro de rango: %d, fuera de rango: %d)\n', ...
        n_test, sum(dentro), sum(fuera));
fprintf('Rango estudiado: [%.0f, %.0f] kW/m^3 en las tres resistencias.\n', ...
        min(Qmin)/1e3, max(Qmax)/1e3);

% 3.PREDICCIÓN CON HOSVD (modelo construido con los 343)
yh  = cell(n_sal,1);   % predicción HOSVD por salida
cfd = cell(n_sal,1);   % valor CFD por salida
for o = 1:n_sal
    Tn = tensores.(col_salidas{o});
    yh{o}  = pred_hosvd(Tn, q1, q2, q3, Qe, r);
    cfd{o} = Te.(col_salidas{o});
end

% 3b.COSTE COMPUTACIONAL: tiempo de predicción HOSVD por caso [s]
% Se construye el modelo HOSVD de cada salida UNA sola vez y se mide el
% tiempo de predecir un caso nuevo (las 4 temperaturas). Los tiempos son muy
% pequeños, por eso se promedian sobre Nrep repeticiones por caso.
MUo = cell(n_sal,1);  Uo = cell(n_sal,1);  CRo = cell(n_sal,1);
tb = tic;
for o = 1:n_sal
    Tn = tensores.(col_salidas{o});  mu = mean(Tn(:));
    [U,~,core] = hosvd3(Tn - mu);
    MUo{o} = mu;  Uo{o} = U;  CRo{o} = core(1:r,1:r,1:r);
end
t_build = toc(tb);

Nrep   = 1000;                 % repeticiones para promediar (tiempos muy pequeños)
t_caso = zeros(n_test,1);
for i = 1:n_test
    tt = tic;
    for k = 1:Nrep
        for o = 1:n_sal
            A = Uo{o}{1}(:,1:r);  B = Uo{o}{2}(:,1:r);  C = Uo{o}{3}(:,1:r);
            u1 = interp1(q1, A, Qe(i,1), 'pchip');
            u2 = interp1(q2, B, Qe(i,2), 'pchip');
            u3 = interp1(q3, C, Qe(i,3), 'pchip');
            val = MUo{o} + reconstruct3d(CRo{o}, u1, u2, u3);
        end
    end
    t_caso(i) = toc(tt) / Nrep;     % tiempo de predecir un caso (4 salidas) [s]
end
fprintf('\nTIEMPO DE PREDICCIÓN HOSVD POR CASO [s] (4 salidas por caso)\n');
fprintf('Construcción del modelo (1 vez, 4 salidas): %.4g s\n', t_build);
fprintf('%-12s %16s\n', 'Caso', 't_prediccion [s]');
for i = 1:n_test
    fprintf('%-12s %16.3e\n', string(Te.Name(i)), t_caso(i));
end
fprintf('Media por caso: %.3e s  (%.4g ms)\n', mean(t_caso), mean(t_caso)*1e3);

% 4.MÉTRICAS POR GRUPO (HOSVD): [rmse mae max r2]
grupos = {'DENTRO de rango','FUERA de rango'};
gmask  = {dentro, fuera};
metG   = zeros(2, n_sal, 4);
for gi = 1:2
    msk = gmask{gi};
    for o = 1:n_sal
        [metG(gi,o,1),metG(gi,o,2),metG(gi,o,3),metG(gi,o,4)] = ...
            metricas(cfd{o}(msk), yh{o}(msk));
    end
end

% 5.RESUMEN POR CONSOLA (HOSVD)
fprintf('\nVALIDACIÓN HOSVD - CASOS NUEVOS (modelo con 343, r=%d)\n', r);
for gi = 1:2
    fprintf('\n%s (%d casos)\n', grupos{gi}, sum(gmask{gi}));
    fprintf('%-18s %12s %12s %12s\n','Salida','RMSE[K]','max[K]','R2');
    for o = 1:n_sal
        fprintf('%-18s %12.4f %12.4f %12.5f\n', col_salidas{o}, ...
            metG(gi,o,1), metG(gi,o,3), metG(gi,o,4));
    end
end
o_mx = find(strcmp(col_salidas,'T_max_K'));  if isempty(o_mx), o_mx = 3; end
fprintf('\n-- Detalle HOSVD en %s (todos los casos) --\n', col_salidas{o_mx});
fprintf('%-12s %8s %10s %10s %9s\n','Caso','Rango','CFD[K]','HOSVD[K]','err[K]');
for i = 1:n_test
    etiqueta = 'dentro';  if fuera(i), etiqueta = 'FUERA'; end
    fprintf('%-12s %8s %10.2f %10.2f %9.2f\n', string(Te.Name(i)), etiqueta, ...
        cfd{o_mx}(i), yh{o_mx}(i), yh{o_mx}(i)-cfd{o_mx}(i));
end

% 6.FIGURAS (HOSVD)
% azul = dentro de rango ; ámbar = fuera de rango.
c_dentro = [0.00 0.45 0.74];   % azul
c_fuera  = [0.93 0.69 0.13];   % ámbar/dorado
errH   = abs(yh{o_mx} - cfd{o_mx});   % |error| HOSVD en T_max
idx_d  = find(dentro);   errH_d = errH(dentro);

% Figura 1: CFD vs predicho (HOSVD), DENTRO de rango (2x2)
figure('Name','HOSVD DENTRO de rango: CFD vs predicho','Color','w','Position',[50 80 950 700]);
for o = 1:n_sal
    subplot(2,2,o);
    yt = cfd{o}(dentro);  yp = yh{o}(dentro);
    plot(yt, yp, 'o', 'MarkerSize',6, 'MarkerFaceColor',c_dentro, 'MarkerEdgeColor',c_dentro); hold on;
    lims = [min([yt;yp]) max([yt;yp])];  plot(lims, lims, 'k--','LineWidth',1.1);
    grid on; axis tight; xlabel('CFD [K]'); ylabel('HOSVD [K]');
    title(col_salidas{o},'Interpreter','none');
    if o==1, legend({'HOSVD','y = x (ideal)'},'Location','northwest','FontSize',8); end
end

% Figura 2: CFD vs predicho (HOSVD), FUERA de rango (2x2)
figure('Name','HOSVD FUERA de rango: CFD vs predicho','Color','w','Position',[70 60 980 720]);
cn = find(fuera);
for o = 1:n_sal
    subplot(2,2,o);
    yt = cfd{o}(fuera);  yp = yh{o}(fuera);
    errs = abs(yp - yt);
    thr  = 3*median(errs);
    shown = errs <= thr;
    if nnz(shown) < 3
        [~,si] = sort(errs);  shown(:) = false;  shown(si(1:min(3,numel(si)))) = true;
    end
    vy = [yt(shown); yp(shown)];
    lo = min(vy);  hi = max(vy);  rng = hi - lo;  if rng <= 0, rng = max(abs(hi),1); end
    W  = [lo - 0.15*rng, hi + 0.15*rng];          % ventana centrada en el grueso de casos
    inw = yp >= W(1) & yp <= W(2);
    hiM = yp > W(2);   loM = yp < W(1);
    h1 = plot(yt(inw), yp(inw), 'd', 'MarkerSize',7, 'MarkerFaceColor',c_fuera, 'MarkerEdgeColor','k'); hold on;
    hL = plot(W, W, 'k--', 'LineWidth',1.1);
    hHi = plot(yt(hiM), repmat(W(2),sum(hiM),1), '^', 'MarkerSize',9, 'MarkerFaceColor',c_fuera, 'MarkerEdgeColor','k');
    plot(yt(loM), repmat(W(1),sum(loM),1), 'v', 'MarkerSize',9, 'MarkerFaceColor',c_fuera, 'MarkerEdgeColor','k');
    for t = find(hiM)'
        text(yt(t), W(2), sprintf(' %d', cn(t)), 'VerticalAlignment','top',    'FontSize',8);
    end
    for t = find(loM)'
        text(yt(t), W(1), sprintf(' %d', cn(t)), 'VerticalAlignment','bottom', 'FontSize',8);
    end
    xlim(W); ylim(W); axis square; grid on;
    xlabel('CFD [K]'); ylabel('HOSVD [K]'); title(col_salidas{o},'Interpreter','none');
    if o==1, legend([h1 hL hHi], {'HOSVD','y = x (ideal)','fuera de escala'}, 'Location','northwest','FontSize',8); end
end

% Figura 3: |error| de T_max por caso, TODOS (azul/ámbar, log)
figure('Name','HOSVD: |error| T_max por caso (todos)','Color','w','Position',[90 70 1000 470]);
b = bar(1:n_test, errH, 0.7, 'FaceColor','flat');
for i = 1:n_test
    if fuera(i), b.CData(i,:) = c_fuera; else, b.CData(i,:) = c_dentro; end
end
set(gca,'YScale','log','FontSize',9); grid on; hold on;
for i = 1:n_test
    text(i, errH(i)*1.4, sprintf('%.3g', errH(i)), ...
        'HorizontalAlignment','center','VerticalAlignment','bottom','FontSize',7);
end
ylim([min(errH)/3, max(errH)*7]);
xlabel('Caso de prueba'); ylabel('|error| T_{max} [K]'); xticks(1:n_test);
hd = patch(NaN,NaN,c_dentro,'EdgeColor','none');
hf = patch(NaN,NaN,c_fuera, 'EdgeColor','none');
legend([hd hf], {'dentro de rango','fuera de rango'}, 'Location','northwest','FontSize',9);

% Figura 4: |error| de T_max por caso, SOLO DENTRO (mismo formato)
figure('Name','HOSVD: |error| T_max por caso (solo dentro)','Color','w','Position',[110 90 820 470]);
bd = bar(1:numel(idx_d), errH_d, 0.7, 'FaceColor',c_dentro, 'EdgeColor','k', 'LineWidth',0.5);
set(gca,'YScale','log','FontSize',9); grid on; hold on;
for j = 1:numel(idx_d)
    text(j, errH_d(j)*1.4, sprintf('%.3g', errH_d(j)), ...
        'HorizontalAlignment','center','VerticalAlignment','bottom','FontSize',8);
end
ylim([min(errH_d)/3, max(errH_d)*7]);
xticks(1:numel(idx_d)); xticklabels(string(idx_d));
xlabel('Caso de prueba (dentro de rango)'); ylabel('|error| T_{max} [K]');

% Figura 5: error máximo por salida, DENTRO vs FUERA (log, etiquetas)
Edentro = reshape(metG(1,:,3), n_sal, 1);
Efuera  = reshape(metG(2,:,3), n_sal, 1);
EmaxDF  = [Edentro, Efuera];
figure('Name','HOSVD: error maximo por salida (dentro vs fuera)','Color','w','Position',[130 70 900 480]);
hb = bar(EmaxDF);
hb(1).FaceColor = c_dentro;  hb(2).FaceColor = c_fuera;
set(gca,'YScale','log','XTickLabel',col_salidas,'TickLabelInterpreter','none','FontSize',9);
ylabel('Error maximo absoluto [K]'); grid on; hold on;
ylim([min(EmaxDF(:))/3, max(EmaxDF(:))*40]);
legend({'dentro de rango','fuera de rango'},'Location','northwest','FontSize',9);
for sgi = 1:2
    for o = 1:n_sal
        yv = EmaxDF(o,sgi);
        text(hb(sgi).XEndPoints(o), yv*1.4, sprintf('%.3g', yv), ...
            'HorizontalAlignment','center','VerticalAlignment','bottom','FontSize',8,'FontWeight','bold');
    end
end

% Figura 6: RMSE por salida, DENTRO vs FUERA (log, etiquetas)
RdentroDF = reshape(metG(1,:,1), n_sal, 1);
RfueraDF  = reshape(metG(2,:,1), n_sal, 1);
RmseDF    = [RdentroDF, RfueraDF];
figure('Name','HOSVD: RMSE por salida (dentro vs fuera)','Color','w','Position',[150 60 900 480]);
hbr = bar(RmseDF);
hbr(1).FaceColor = c_dentro;  hbr(2).FaceColor = c_fuera;
set(gca,'YScale','log','XTickLabel',col_salidas,'TickLabelInterpreter','none','FontSize',9);
ylabel('RMSE [K]'); grid on; hold on;
ylim([min(RmseDF(:))/3, max(RmseDF(:))*40]);
legend({'dentro de rango','fuera de rango'},'Location','northwest','FontSize',9);
for sgi = 1:2
    for o = 1:n_sal
        yv = RmseDF(o,sgi);
        text(hbr(sgi).XEndPoints(o), yv*1.4, sprintf('%.3g', yv), ...
            'HorizontalAlignment','center','VerticalAlignment','bottom','FontSize',8,'FontWeight','bold');
    end
end

% Figura 7 (3D): espacio de potencias; color = |error| T_max (escala log)
lo = [min(q1) min(q2) min(q3)]/1e3;   hi = [max(q1) max(q2) max(q3)]/1e3;
figure('Name','HOSVD 3D: espacio de potencias y error','Color','w','Position',[150 80 920 680]);
ex = [lo(1) hi(1)];  ey = [lo(2) hi(2)];  ez = [lo(3) hi(3)];
E = [ ex(1) ey(1) ez(1)  ex(2) ey(1) ez(1);
      ex(2) ey(1) ez(1)  ex(2) ey(2) ez(1);
      ex(2) ey(2) ez(1)  ex(1) ey(2) ez(1);
      ex(1) ey(2) ez(1)  ex(1) ey(1) ez(1);
      ex(1) ey(1) ez(2)  ex(2) ey(1) ez(2);
      ex(2) ey(1) ez(2)  ex(2) ey(2) ez(2);
      ex(2) ey(2) ez(2)  ex(1) ey(2) ez(2);
      ex(1) ey(2) ez(2)  ex(1) ey(1) ez(2);
      ex(1) ey(1) ez(1)  ex(1) ey(1) ez(2);
      ex(2) ey(1) ez(1)  ex(2) ey(1) ez(2);
      ex(2) ey(2) ez(1)  ex(2) ey(2) ez(2);
      ex(1) ey(2) ez(1)  ex(1) ey(2) ez(2) ];
h_box = [];
for k = 1:size(E,1)
    hh = plot3([E(k,1) E(k,4)], [E(k,2) E(k,5)], [E(k,3) E(k,6)], '-', ...
               'Color',[.55 .55 .55], 'LineWidth',1.2); hold on;
    if k==1, h_box = hh; end
end
sc_d = scatter3(Qe(dentro,1)/1e3, Qe(dentro,2)/1e3, Qe(dentro,3)/1e3, 95,  errH(dentro), ...
                'filled','Marker','o','MarkerEdgeColor','k');
sc_f = scatter3(Qe(fuera,1)/1e3,  Qe(fuera,2)/1e3,  Qe(fuera,3)/1e3, 120, errH(fuera), ...
                'filled','Marker','d','MarkerEdgeColor','k');
colormap(parula);  set(gca,'ColorScale','log');  caxis([min(errH) max(errH)]);
cb = colorbar;  cb.Label.String = '|error| T_{max} [K]';
xlabel('Q_1 [kW/m^3]'); ylabel('Q_2 [kW/m^3]'); zlabel('Q_3 [kW/m^3]');
view(135,18); grid on; axis tight;
hp_d = plot3(NaN,NaN,NaN,'o','MarkerFaceColor',[.45 .45 .45],'MarkerEdgeColor','k','MarkerSize',8);
hp_f = plot3(NaN,NaN,NaN,'d','MarkerFaceColor',[.45 .45 .45],'MarkerEdgeColor','k','MarkerSize',8);
legend([hp_d hp_f h_box], {'dentro de rango','fuera de rango','caja = rango estudiado'}, ...
       'Location','southoutside','Orientation','horizontal','FontSize',9);

% 7.GUARDAR RESULTADOS
save(fullfile(carpeta,'resultados_validacion_nuevos.mat'), ...
     'grupos','yh','cfd','metG','col_salidas','Qe','Te','r','dentro','fuera','t_caso','t_build');
fprintf('\nGuardado: resultados_validacion_nuevos.mat\n');
fprintf('Paso 4 OK.\n');

% FUNCIONES LOCALES
function yp = pred_hosvd(Tg, qa, qb, qc, Qev, r)
% HOSVD + interpolación de las matrices de factores (sobre la rejilla 7x7x7).
    mu = mean(Tg(:));
    [U, ~, core] = hosvd3(Tg - mu);
    A = U{1}(:,1:r);  B = U{2}(:,1:r);  C = U{3}(:,1:r);
    cr = core(1:r,1:r,1:r);
    N = size(Qev,1);  yp = zeros(N,1);
    for t = 1:N
        u1 = zeros(1,r); u2 = zeros(1,r); u3 = zeros(1,r);
        for m = 1:r
            u1(m) = interp1(qa, A(:,m), Qev(t,1), 'pchip');
            u2(m) = interp1(qb, B(:,m), Qev(t,2), 'pchip');
            u3(m) = interp1(qc, C(:,m), Qev(t,3), 'pchip');
        end
        val = reconstruct3d(cr, u1, u2, u3);
        yp(t) = mu + val(1);
    end
end

function [rmse, mae, mx, r2] = metricas(y, yp)
    y = y(:);  yp = yp(:);  e = yp - y;
    rmse = sqrt(mean(e.^2));  mae = mean(abs(e));  mx = max(abs(e));
    r2 = 1 - sum(e.^2) / sum((y - mean(y)).^2);
end

function [U, s, core] = hosvd3(T)
    sz = size(T);
    T1 = reshape(T,               sz(1), sz(2)*sz(3));
    T2 = reshape(permute(T,[2 1 3]), sz(2), sz(1)*sz(3));
    T3 = reshape(permute(T,[3 1 2]), sz(3), sz(1)*sz(2));
    [U1,S1,~] = svd(T1,'econ');  [U2,S2,~] = svd(T2,'econ');  [U3,S3,~] = svd(T3,'econ');
    U = {U1, U2, U3};  s = {diag(S1), diag(S2), diag(S3)};
    core = reconstruct3d(T, U1', U2', U3');
end

function Trec = reconstruct3d(core, A, B, C)
    [r1, r2, r3] = tam3(core);
    I = size(A,1);  J = size(B,1);  K = size(C,1);
    M1 = A * reshape(core, r1, r2*r3);
    G  = reshape(M1, I, r2, r3);
    G2 = reshape(permute(G,[2 1 3]), r2, I*r3);
    M2 = B * G2;
    G  = permute(reshape(M2, J, I, r3), [2 1 3]);
    G3 = reshape(permute(G,[3 1 2]), r3, I*J);
    M3 = C * G3;
    Trec = permute(reshape(M3, K, I, J), [2 3 1]);
end

function [a,b,c] = tam3(T)
    sz = size(T);  sz(end+1:3) = 1;
    a = sz(1);  b = sz(2);  c = sz(3);
end
```