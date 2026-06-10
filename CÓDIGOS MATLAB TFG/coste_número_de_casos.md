v b gv

```matlab
% fig_escenarios.m  

clear; close all; clc;

% Figura 1 y Figura 2  (desde n = 0)
fig_escenarios_plot(4, false, 'fig_escenarios.png');
fig_escenarios_plot(0, true , 'fig_escenarios_desde_cero.png');

function fig_escenarios_plot(xmin, drawExt, fname)
    AZUL  = [0 0 1];                     
    tcaso  = 1327.5; % s/caso (media cronometrada)
    k      = tcaso/86400; % factor casos -> dias
    considered = [6 7 8 10];
    DIAS = ['d' char(237) 'as']; % "días" con tilde

    % Etiquetas: {n, N, dx(n), dy(casos), HAlign, VAlign, negrita, texto}
    labels = { ...
      { 6,  216, -0.13,  35, 'right', 'bottom', false, {'n = 6','216 casos',     ['3,3 '  DIAS]} }, ...
      { 7,  343,  0.13, -35, 'left',  'top',    true,  {'n = 7 (adoptado)','343 casos', ['5,3 '  DIAS]} }, ...
      { 8,  512, -0.13,  35, 'right', 'bottom', false, {'n = 8','512 casos',     ['7,9 '  DIAS]} }, ...
      {10, 1000, -0.13,  35, 'right', 'bottom', false, {'n = 10','1000 casos',   ['15,4 ' DIAS]} } ...
    };

    figure('Color','w','Units','centimeters','Position',[2 2 16 11]);
    ax = axes; hold(ax,'on'); box(ax,'on'); grid(ax,'on');
    set(ax,'FontSize',12,'GridAlpha',0.15);

    yyaxis left
    ns = 4:0.02:11;   plot(ns, ns.^3, 'b', 'LineWidth', 1.5); % tramo continuo
    if drawExt
        nd = 0:0.02:4;   plot(nd, nd.^3, 'b--', 'LineWidth', 1.5); % extensión a trazos
    end
    for c = considered
        if c == 7
            plot(c, c^3, 'o', 'MarkerFaceColor', AZUL, 'MarkerEdgeColor', AZUL, 'MarkerSize', 7.5);
        else
            plot(c, c^3, 'o', 'MarkerFaceColor', 'w', 'MarkerEdgeColor', AZUL, 'MarkerSize', 7, 'LineWidth', 1.6);
        end
    end
    for idx = 1:numel(labels)
        L = labels{idx};
        t = text(L{1}+L{3}, L{2}+L{4}, L{8}, 'FontSize', 11, ...
                 'HorizontalAlignment', L{5}, 'VerticalAlignment', L{6}, 'Color', 'k');
        if L{7}
            set(t, 'FontWeight', 'bold');
        end
    end
    ylabel(['N' char(250) 'mero de casos, N = n^{3}'], 'FontSize', 13);
    xlim([xmin 11]); ylim([0 1380]); xticks(xmin:11);
    ax.YAxis(1).Color = 'k';

    yyaxis right
    ylim([0 1380*k]);
    yticks(0:2.5:20);
    yticklabels({'0','2,5','5','7,5','10','12,5','15','17,5','20'});
    ylabel(['Tiempo de c' char(225) 'lculo estimado (' DIAS ')'], 'FontSize', 13);
    ax.YAxis(2).Color = 'k';
    yyaxis left

    xlabel('Niveles por resistencia, n', 'FontSize', 13);

    exportgraphics(gcf, fname, 'Resolution', 300);   
end
```