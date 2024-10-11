# Questao2-Prova-EspecialistaBI

# Sobre a questão

Contexto do Mercado: Em um sistema de e-commerce, produtos são vendidos em múltiplas plataformas simultaneamente. Ocorre um problema frequente de conflito de estoque, onde o mesmo produto é vendido em duas plataformas quase ao mesmo tempo, levando a vendas que não podem ser atendidas devido à falta de estoque.

Desafio: Implementar uma solução de banco de dados que minimize conflitos de estoque, garantindo que as atualizações de inventário sejam processadas em tempo real e de forma que vendas simultâneas não resultem em comprometimento do estoque disponível.

Crie as Tabelas e Dados:

Tabela de Produtos: Uma tabela contendo ID do Produto, Nome do Produto, Quantidade em Estoque. Essa tabela será utilizada
para simular o estoque atual de produtos disponíveis para venda.
Tabela de Vendas: Uma tabela para registrar cada venda, contendo ID da Venda, ID do Produto, Quantidade Vendida, Timestamp da Venda. Essa tabela simula vendas ocorrendo em diferentes momentos.

# Código utilizado para solução

```SQL

-- 1. Remover tabelas existentes (se houver)
IF OBJECT_ID('Vendas', 'U') IS NOT NULL
    DROP TABLE Vendas;

IF OBJECT_ID('Produtos', 'U') IS NOT NULL
    DROP TABLE Produtos;

-- 2. Criação das Tabelas
CREATE TABLE Produtos (
    ProdutoID INT PRIMARY KEY,
    NomeProduto NVARCHAR(100) NOT NULL,
    QuantidadeEstoque INT NOT NULL CHECK (QuantidadeEstoque >= 0)
);

CREATE TABLE Vendas (
    VendaID INT IDENTITY PRIMARY KEY,
    ProdutoID INT NOT NULL,
    QuantidadeVendida INT NOT NULL CHECK (QuantidadeVendida > 0),
    DataHoraVenda DATETIME DEFAULT GETDATE(),
    CONSTRAINT FK_ProdutoVenda FOREIGN KEY (ProdutoID) REFERENCES Produtos(ProdutoID)
);

-- 3. Inserção de Dados de Exemplo
INSERT INTO Produtos (ProdutoID, NomeProduto, QuantidadeEstoque)
VALUES 
(1, 'Produto A', 50),
(2, 'Produto B', 30),
(3, 'Produto C', 20);

-- 4. Criação do Procedimento Armazenado
IF OBJECT_ID('RegistrarVenda', 'P') IS NOT NULL
    DROP PROCEDURE RegistrarVenda;
GO

CREATE PROCEDURE RegistrarVenda
    @ProdutoID INT,
    @QuantidadeVendida INT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRANSACTION;

    BEGIN TRY
        -- Verifica se há estoque suficiente e bloqueia a linha para evitar atualizações concorrentes
        UPDATE Produtos WITH (UPDLOCK, ROWLOCK)
        SET QuantidadeEstoque = QuantidadeEstoque - @QuantidadeVendida
        WHERE ProdutoID = @ProdutoID 
          AND QuantidadeEstoque >= @QuantidadeVendida;

        -- Verifica se a atualização de estoque ocorreu
        IF @@ROWCOUNT = 0
        BEGIN
            -- Se não houve atualização, significa que o estoque não era suficiente
            ROLLBACK TRANSACTION;
            PRINT 'Estoque insuficiente';
        END
        ELSE
        BEGIN
            -- Registra a venda
            INSERT INTO Vendas (ProdutoID, QuantidadeVendida)
            VALUES (@ProdutoID, @QuantidadeVendida);

            COMMIT TRANSACTION;
            PRINT 'Venda realizada com sucesso';
        END;
    END TRY
    BEGIN CATCH
        -- Em caso de erro, reverte a transação e exibe a mensagem de erro
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        PRINT 'Erro: ' + @ErrorMessage;
    END CATCH;
END;
GO

-- 5. Testar o Procedimento Armazenado
EXEC RegistrarVenda @ProdutoID = 1, @QuantidadeVendida = 5;

-- 6. Verificar os Resultados
SELECT * FROM Produtos WHERE ProdutoID = 1;
SELECT * FROM Vendas WHERE ProdutoID = 1;

